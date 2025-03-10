<template>

  <div class="full-page">
    <UiAlert
      v-if="coreError"
      :dismissible="false"
      class="alert"
      type="error"
    >
      <span>{{ coreError }}</span><br>
      <KButton
        v-if="restart"
        appearance="basic-link"
        :text="coreString('startOverAction')"
        @click="startOver"
      />
    </UiAlert>
    <main class="content">
      <KolibriLoadingSnippet />
      <h1 class="page-title">
        {{ $tr('pageTitle') }}
      </h1>
      <p class="message">
        {{ $tr('pleaseWaitMessage') }}
      </p>
    </main>
  </div>

</template>


<script>

  import omitBy from 'lodash/omitBy';
  import get from 'lodash/get';
  import { currentLanguage } from 'kolibri.utils.i18n';
  import { checkCapability } from 'kolibri.utils.appCapabilities';
  import redirectBrowser from 'kolibri.utils.redirectBrowser';
  import KolibriLoadingSnippet from 'kolibri.coreVue.components.KolibriLoadingSnippet';
  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import UiAlert from 'kolibri-design-system/lib/keen/UiAlert';
  import urls from 'kolibri.urls';
  import client from 'kolibri.client';
  import Lockr from 'lockr';
  import { DeviceTypePresets, UsePresets } from '../../constants';

  export default {
    name: 'SettingUpKolibri',
    components: { UiAlert, KolibriLoadingSnippet },
    inject: ['wizardService'],
    mixins: [commonCoreStrings],
    data() {
      return {
        restart: false,
      };
    },
    computed: {
      coreError() {
        if (this.$store) {
          return this.$store.state.core.error;
        } else {
          return null;
        }
      },
      facilityData() {
        const usersName = get(this.wizardContext('superuser'), 'full_name', '');
        const facilityName =
          this.wizardContext('facilityName') ||
          // full names may be up to 120 chars, but facility names can only be 100 chars
          this.$tr('onMyOwnFacilityName', { name: usersName }).slice(0, 100);
        const selectedFacility = this.wizardContext('selectedFacility');
        if (selectedFacility) {
          if (selectedFacility.id) {
            // Imported a facility already otherwise we wouldn't have an ID yet,
            // so we'll just be sending off the `facility_id`
            return { facility_id: selectedFacility.id };
          } else {
            // Otherwise we'll pass the facility data we have (including settings set by user)
            return { facility: selectedFacility };
          }
        } else {
          return { facility: { name: facilityName } };
        }
      },
      learnerCanLoginWithNoPassword() {
        // The user answers the question "Enable passwords?" -- so the `requirePassword` value
        // is the boolean opposite of whatever the value we need to assign to
        // `learner_can_login_with_no_password` in the API call.
        // If there is already a facility imported, we will use its value
        // If it is `null`, then it was never set by the user and we set to require passwords
        const { facility, facility_id } = this.facilityData;
        // If we have a facility_id then we imported the facility
        if (facility_id) {
          return null;
        }
        const facilitySetting = get(facility, 'learner_can_login_with_no_password', null);
        if (facilitySetting !== null) {
          return facilitySetting;
        } else {
          return this.wizardContext('requirePassword') === null
            ? true
            : !this.wizardContext('requirePassword');
        }
      },
      learnerCanEditPassword() {
        // Note that we don't ask this question of a user during onboarding -- however,
        // the nonformal facility will set this to `true` by default -- which does not jive
        // with the possibility that a user can login with no password
        if (
          // Learner cannot edit a password they cannot set
          this.learnerCanLoginWithNoPassword ||
          // OS on my own users don't use password to sign in
          (this.isOnMyOwnSetup && checkCapability('get_os_user'))
        ) {
          return false; // Learner cannot edit a password they cannot set
        } else {
          return null; // We'll set this to a key and null values are removed from the API call
        }
      },
      /** The data we will use to initialize the device during provisioning */
      deviceProvisioningData() {
        let superuser = null;
        // We only need a superuser if we cannot get the OS user; null valued keys will be omitted
        // in the eventual API call
        if (!checkCapability('get_os_user')) {
          // Here we see if we've set a firstImportedLodUser -- if they exist, they must be the
          // superuser as they were the first imported user.
          if (this.wizardContext('firstImportedLodUser')) {
            superuser = this.wizardContext('firstImportedLodUser');
          }
          if (!superuser) {
            // If we are creating a user, their data is in the Vuex store because UserCredentials is
            // tightly coupled to it (for now).
            superuser = this.wizardContext('superuser') || this.$store.state.onboardingData.user;
          }
        }

        const settings = {
          learner_can_login_with_no_password: this.learnerCanLoginWithNoPassword,
          learner_can_edit_password: this.learnerCanEditPassword,
          on_my_own_setup: this.isOnMyOwnSetup,
          learner_can_sign_up: this.wizardContext('learnerCanCreateAccount'),
        };

        const payload = {
          ...this.facilityData,
          superuser,
          settings: omitBy(settings, v => v === null),
          preset: this.wizardContext('formalOrNonformal') || 'nonformal',
          language_id: currentLanguage,
          device_name:
            this.wizardContext('deviceName') ||
            // full names may be up to 120 chars, but device names can only be 50 chars
            this.$tr('onMyOwnDeviceName', { name: get(superuser, 'full_name', '') }).slice(0, 50),
          allow_guest_access: Boolean(this.wizardContext('guestAccess')),
          is_provisioned: true,
          os_user: checkCapability('get_os_user'),
          is_soud: this.wizardContext('fullOrLOD') === DeviceTypePresets.LOD,
        };

        // Remove anything that is `null` value
        return omitBy(payload, v => v === null);
      },

      /** Introspecting the machine via it's `state.context` properties */
      isOnMyOwnSetup() {
        return this.wizardContext('onMyOwnOrGroup') == UsePresets.ON_MY_OWN;
      },
    },
    mounted() {
      this.provisionDevice();
    },
    methods: {
      startOver() {
        this.$store.dispatch('clearError');
        this.wizardService.send('START_OVER');
      },
      // A helper for readability
      wizardContext(key) {
        return this.wizardService.state.context[key];
      },
      provisionDevice() {
        client({
          url: urls['kolibri:core:deviceprovision'](),
          method: 'POST',
          data: this.deviceProvisioningData,
        })
          .then(() => {
            const welcomeDismissalKey = 'DEVICE_WELCOME_MODAL_DISMISSED';
            window.sessionStorage.setItem(welcomeDismissalKey, false);

            Lockr.rm('savedState'); // Clear out saved state machine
            redirectBrowser();
          })
          .catch(e => {
            this.restart = e.response.status === 400;
            this.$store.dispatch('handleApiError', { error: e });
          });
      },
    },
    $trs: {
      pageTitle: {
        message: 'Setting up Kolibri',
        context: 'The title of the page',
      },
      pleaseWaitMessage: {
        message: 'This may take several minutes',
        context: 'Kolibri is working in the background and the user may need to wait',
      },
      onMyOwnDeviceName: {
        message: 'Personal device for {name}',
        context:
          "The default device name for a user installing Kolibri using the personal 'on my own' (formerly Quick Start) flow",
      },
      onMyOwnFacilityName: {
        message: 'Home Facility for {name}',
        context:
          "Default facility name when Kolibri is installed with the 'Quick start' setup option for at home learning, outside any type of structure or institution like a school or a library. '{name}' will display the full name of the super admin user for their Kolibri server. Note that users can change this default name after the setup, and put whatever name they want to use for their home facility.",
      },
    },
  };

</script>


<style scoped lang="scss">

  .alert {
    position: relative;
    top: 0;
    left: 0;
    margin: 16px;
    text-align: left;
  }

  .full-page {
    /* Fill the screen, no scroll bars */
    position: relative;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
  }

  .content {
    /* Vertically centered */
    position: relative;
    top: 50%;
    transform: translateY(-50%);
  }

  .page-title,
  .message {
    padding: 0 1em;
    text-align: center;
  }

  .page-title {
    font-size: 1.5em;
  }

</style>
