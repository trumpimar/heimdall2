<template>
  <v-stepper v-model="step" vertical class="elevation-0">
    <v-stepper-step :complete="!!assumedRole" step="1">
      Account Credentials
    </v-stepper-step>

    <AuthStepBasic
      :access-token.sync="accessToken"
      :secret-token.sync="secretToken"
      :region.sync="region"
      @auth-basic="handle_basic"
      @goto-mfa="handle_goto_mfa"
      @show-help="showHelp = true"
    />

    <v-stepper-step :complete="!!assumedRole && assumedRole.from_mfa" step="2">
      MFA Authorization
    </v-stepper-step>

    <AuthStepMFA
      :mfa-token.sync="mfaToken"
      :mfa-serial.sync="mfaSerial"
      @auth-mfa="handle_proceed_mfa"
      @exit-mfa="handle_cancel_mfa"
    />

    <v-stepper-step step="3"> Browse Bucket </v-stepper-step>

    <FileList
      :auth="assumedRole"
      :files="files"
      @exit-list="handle_cancel_mfa"
      @got-files="got_files"
      @load-bucket="load_bucket"
    />
    <v-overlay :opacity="50" absolute="absolute" :value="showHelp">
      <div class="text-center">
        <p>
          <span>
            For installation instructions and further information, check here:
          </span>
          <v-btn
            target="_blank"
            href="https://github.com/mitre/heimdall2#aws-s3"
            text
            color="info"
            px-0
          >
            <v-icon pr-2>mdi-github-circle</v-icon>
            S3 Configuration
          </v-btn>
        </p>
        <v-btn color="info" @click="showHelp = false"> Close </v-btn>
      </div>
    </v-overlay>
  </v-stepper>
</template>

<script lang="ts">
import AuthStepBasic from '@/components/global/upload_tabs/aws/AuthStepBasic.vue';
import AuthStepMFA from '@/components/global/upload_tabs/aws/AuthStepMFA.vue';
import FileList from '@/components/global/upload_tabs/aws/FileList.vue';
import {FileID} from '@/store/report_intake';
import {SnackbarModule} from '@/store/snackbar';
import {
  Auth,
  AUTH_DURATION,
  get_session_token,
  MFAInfo,
  transcribe_error
} from '@/utilities/aws_util';
import {LocalStorageVal} from '@/utilities/helper_util';
import S3 from 'aws-sdk/clients/s3';
import {AWSError} from 'aws-sdk/lib/error';
import Vue from 'vue';
import Component from 'vue-class-component';

/** The cached session info */
const localSessionInformation = new LocalStorageVal<Auth | null>(
  'aws_session_info'
);

/**
 * Uploads data to the store with unique IDs asynchronously as soon as data is entered.
 * Emits "got-files" with a list of the unique_ids of the loaded files.
 */
@Component({
  components: {
    AuthStepBasic,
    AuthStepMFA,
    FileList
  }
})
export default class S3Reader extends Vue {
  /** Form required field rules. Maybe eventually expand to other stuff */
  reqRule = (v: string | null | undefined) =>
    (v || '').trim().length > 0 || 'Field is Required';

  /** Passed from step 1 to step 2 (MFA) if necessary */
  /** State of all globally relevant fields */
  accessToken = '';
  secretToken = '';
  region = '';
  mfaSerial = '';
  mfaToken = '';
  showHelp = false;

  /** Our session information, generated by AWS STS */
  assumedRole: Auth | null = null;

  /** Current step */
  step = 1;

  /** Currently loaded file list from bucket */
  files: S3.Object[] = [];

  /**
   * Handle a basic login.
   * Gets a session token
   */
  handle_basic() {
    // Attempt to assume role based on if we've determined 2fa necessary
    get_session_token(
      this.accessToken,
      this.secretToken,
      this.region,
      AUTH_DURATION
    ).then(
      // Success of get session token - now need to determine if MFA necessary
      (success) => {
        this.assumedRole = success;
        this.step = 3;
      },

      // Failure of initial get session token: want to set error normally
      (failure) => {
        this.handle_error(failure);
      }
    );
  }

  /** If the user tries to login by going to MFA, first check that the account is valid */
  handle_goto_mfa() {
    // Attempt to assume role based on if we've determined 2fa necessary
    // Don't need the duration to be very long
    get_session_token(this.accessToken, this.secretToken, this.region, 10).then(
      // Success of get session token - now need to determine if MFA necessary
      () => {
        this.step = 2;
      },

      // Failure of initial get session token: want to set error normally
      (failure) => {
        this.handle_error(failure);
      }
    );
  }

  handle_cancel_mfa() {
    this.step = 1;
    this.mfaToken = '';
    this.assumedRole = null; // Just in case
  }

  handle_exit_list() {
    this.step = 1;
    this.mfaToken = '';
    this.assumedRole = null;
    this.files = []; // Just in case
  }

  /** Handle an MFA login.
   * Determine whether further action is necessary
   */
  handle_proceed_mfa() {
    // Build our mfa params
    const mfa: MFAInfo = {
      SerialNumber: this.mfaSerial || null,
      TokenCode: this.mfaToken
    };

    // Attempt to assume role based on if we've determined 2fa necessary
    get_session_token(
      this.accessToken,
      this.secretToken,
      this.region,
      AUTH_DURATION,
      mfa
    ).then(
      (success) => {
        // Keep them
        this.assumedRole = success;
        this.step = 3;
      },
      (failure) => {
        this.handle_error(failure);
      }
    );
  }

  /** On mount, try to look up stored auth info */
  mounted() {
    // Load our session, if there is one
    this.assumedRole = localSessionInformation.getDefault(null);
    if (this.assumedRole) {
      this.step = 3;
    }
  }

  /** Attempt to load.
   * Basically just wraps fetch_files with error handling
   */
  async load_bucket(name: string) {
    const s3 = new S3({
      ...this.assumedRole!.creds,
      region: this.assumedRole!.region || 'us-east-1'
    });
    await s3
      .listObjectsV2({
        Bucket: name,
        MaxKeys: 100
      })
      .promise()
      .then((success) => {
        this.files = success.Contents || [];
      })
      .catch((failure) => this.handle_error(failure));
  }

  /** Save the current credentials to local storage */
  save_creds() {
    localSessionInformation.set(this.assumedRole);
  }

  /** Callback to handle an AWS error.
   * Sets shown error.
   */
  handle_error(error: AWSError): void {
    const formattedError = transcribe_error(error);
    // Toast whatever error we got
    SnackbarModule.failure(formattedError);
  }

  /** Callback on got files */
  got_files(files: Array<FileID>) {
    this.$emit('got-files', files);
  }
}
</script>
