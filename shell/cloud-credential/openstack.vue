<script>
import Banner from '@components/Banner/Banner.vue';
import { LabeledInput } from '@components/Form/LabeledInput';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { parse as parseUrl } from '@shell/utils/url';
import { _CREATE } from '@shell/config/query-params';
import BusyButton from '@shell/components/BusyButton.vue';
import { Openstack } from '@shell/utils/openstack.ts';
import { Checkbox } from '@components/Form/Checkbox';

export default {
  components: {
    Banner,
    BusyButton,
    Checkbox,
    LabeledInput,
    LabeledSelect,
  },

  props: {
    mode: {
      type:     String,
      required: true,
    },

    value: {
      type:     Object,
      required: true,
    },
  },

  async fetch() {
    this.driver = await this.$store.dispatch('rancher/find', {
      type: 'nodedriver',
      id:   'openstack'
    });
  },

  data() {
    if (this.mode !== _CREATE) {
      this.value.decodedData.useAppCred = this.value.annotations['openstack.cattle.io/useAppCred'] === 'true';
    }

    return {
      projects:       null,
      regions:        null,
      step:           1,
      busy:           false,
      project:        '',
      region:         '',
      errorAllowHost: false,
      driver:         {},
      allowBusy:      false,
      error:          '',
    };
  },

  computed: {
    projectOptions() {
      const sorted = (this.projects || []).sort((a, b) => a.name.localeCompare(b.name));

      return sorted.map((p) => {
        return {
          label: p.name,
          value: p.id
        };
      });
    },

    regionOptions() {
      const sorted = (this.regions || []).sort((a, b) => a.id.localeCompare(b.id));

      let regs = sorted.map((p) => {
        return {
          label: p.id,
          value: p.id
        };
      });
      regs.push({
        label: 'None',
        value: ''
      });
      return regs;
    },

    hostname() {
      const u = parseUrl(this.value.decodedData.authUrl);

      return u?.host || '';
    },

    canAuthenticate() {
      if (this.value?.decodedData?.useAppCred) {
        return !!this.value?.decodedData?.authUrl &&
          !!this.value?.decodedData?.domainName &&
          !!this.value?.decodedData?.applicationCredentialId &&
          !!this.value?.decodedData?.applicationCredentialSecret;
      }
      return !!this.value?.decodedData?.authUrl &&
        !!this.value?.decodedData?.domainName &&
        !!this.value?.decodedData?.username &&
        !!this.value?.decodedData?.password;
    }
  },

  created() {
    this.$emit('validationChanged', false);
  },

  methods: {
    // TODO: Validate that we can get a token for the project that the user has selected
    test() {
      // In the cluster creation flow, annotations is not set, so ensure it is set first
      this.value.annotations = this.value.annotations || {};

      this.value.annotations['openstack.cattle.io/useAppCred'] = this.value.decodedData.useAppCred;

      const project = this.projects.find(p => p.id === this.project);

      if (project) {
        this.value.setData('tenantName', project.name);
        this.value.setData('tenantDomainName', project.domain_id);
      }

      this.value.setData('region', this.region);

      return true;
    },

    // when the user clicked 'edit auth config', clear the projects and set the step back to 1
    // so the user can modify the credentials needed to fetch the projects
    clear() {
      this.step = 1;
      this.projects = null;
      this.errorAllowHost = false;

      // Tell parent that the form is not invalid
      this.$emit('validationChanged', false);
    },

    hostInAllowList() {
      if (!this.driver?.whitelistDomains) {
        return false;
      }

      const u = parseUrl(this.value.decodedData.authUrl);

      if (!u.host) {
        return true;
      }

      return (this.driver?.whitelistDomains || []).includes(u.host);
    },

    async addHostToAllowList() {
      this.allowBusy = true;
      const u = parseUrl(this.value.decodedData.authUrl);

      this.driver.whitelistDomains = this.driver.whitelistDomains || [];

      if (!this.hostInAllowList()) {
        this.driver.whitelistDomains.push(u.host);
      }

      try {
        await this.driver.save();

        this.$refs.connect.$el.click();
      } catch (e) {
        console.error('Could not update driver', e); // eslint-disable-line no-console
        this.allowBusy = false;
      }
    },

    async connect(cb) {
      this.error = '';
      this.errorAllowHost = false;

      let okay = false;

      if (!this.value.decodedData.authUrl) {
        return cb(okay);
      }

      const os = new Openstack(this.$store, {
        endpoint:   this.value.decodedData.authUrl, 
        domainName: this.value.decodedData.domainName,
        username:   this.value.decodedData.username,
        password:   this.value.decodedData.password,
        appCredId: this.value.decodedData.applicationCredentialId,
        appCredSecret: this.value.decodedData.applicationCredentialSecret,
        useAppCred: this.value.decodedData.useAppCred,
      });

      this.allowBusy = false;
      this.step = 2;
      this.busy = true;

      const res = await os.getToken();

      if (res.error) {
        console.error(res.error); // eslint-disable-line no-console
        okay = false;

        this.step = 1;
        this.projects = null;

        if (res.error._status === 502 && !this.hostInAllowList()) {
          this.errorAllowHost = true;
        } else {
          if (res.error._status === 502) {
            // Still got 502, even with URL in the allow list
            this.error = this.t('cluster.credential.openstack.auth.errors.badGateway');
          } else if (res.error._status === 401) {
            this.error = this.t('cluster.credential.openstack.auth.errors.unauthorized');
          } else {
            // Generic error
            this.error = res.error.message || this.t('cluster.credential.openstack.auth.errors.other');
          }
        }
      } else {
        const projects = await os.getProjects();

        if (!projects.error) {
          this.projects = projects;
        } else {
          this.error = projects.error.message;
        }

        const regions = await os.getRegions();

        if (!regions.error) {
          this.regions = regions;
          okay = true;
        } else {
          // Could not list regions, so infer them from the project
          const prj = this.projectOptions[0].value;
          const project = this.projects.find(p => p.id === prj);

          if (project) {
            const osRegions = new Openstack(this.$store, {
              ...os,
              projectName: project.name,
              projectId: project.id,
              projectDomainName: project.domain_id
            });

            // Fetch a token with the project, so we get the endpoint catalog
            await osRegions.getToken().then(() => {
              this.regions = osRegions.regionsFromCatalog;
            });
          }

          // this.error', regions.error.message || this.t('cluster.credential.openstack.auth.errors.regions'));
        }
      }
      this.busy = false;
      this.project = this.projectOptions[0]?.value;
      this.region = this.regionOptions[0]?.value;
      okay = true;
      this.$emit('validationChanged', okay);

      cb(okay);
    }
  }
};
</script>

<template>
  <div>
    <div class="row">
      <div class="col span-6">
        <LabeledInput
          :value="value.decodedData.authUrl"
          :disabled="step !== 1"
          label-key="cluster.credential.openstack.auth.fields.endpoint"
          placeholder-key="cluster.credential.openstack.auth.placeholders.endpoint"
          type="text"
          :mode="mode"
          @update:value="value.setData('authUrl', $event);"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          :value="value.decodedData.domainName"
          :disabled="step !== 1"
          label-key="cluster.credential.openstack.auth.fields.domainName"
          placeholder-key="cluster.credential.openstack.auth.placeholders.domainName"
          type="text"
          :mode="mode"
          @update:value="value.setData('domainName', $event);"
        />
      </div>
    </div>
     <div class="row">
      <div class="col span-6">
       <Checkbox
        :mode="mode"
        class="mt-20"
        :value="value.decodedData.useAppCred"
        label-key="cluster.credential.openstack.auth.fields.useAppCred"
         :disabled="step !== 1"
        @update:value="value.setData('useAppCred', $event);"
      />
      </div>
    </div>
   <div class="row">
      <div class="col span-6">
        <LabeledInput
          v-if="!value.decodedData.useAppCred"
          :value="value.decodedData.username"
          :disabled="step !== 1"
          class="mt-20"
          label-key="cluster.credential.openstack.auth.fields.username"
          placeholder-key="cluster.credential.openstack.auth.placeholders.username"
          type="text"
          :mode="mode"
          @update:value="value.setData('username', $event);"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-if="!value.decodedData.useAppCred"
          :value="value.decodedData.password"
          :disabled="step !== 1"
          class="mt-20"
          label-key="cluster.credential.openstack.auth.fields.password"
          placeholder-key="cluster.credential.openstack.auth.placeholders.password"
          type="password"
          :mode="mode"
          @update:value="value.setData('password', $event);"
        />
      </div> 
    </div>
    <div class="row">
      <div class="col span-6">
        <LabeledInput
          v-if="value.decodedData.useAppCred"
          :value="value.decodedData.applicationCredentialId"
          :disabled="step !== 1"
          class="mt-20"
          label-key="cluster.credential.openstack.auth.fields.appCredId"
          placeholder-key="cluster.credential.openstack.auth.placeholders.appCredId"
          type="text"
          :mode="mode"
          @update:value="value.setData('applicationCredentialId', $event);"
        />
      </div> 
      <div class="col span-6">
        <LabeledInput
          v-if="value.decodedData.useAppCred"
          :value="value.decodedData.applicationCredentialSecret"
          :disabled="step !== 1"
          class="mt-20"
          label-key="cluster.credential.openstack.auth.fields.appCredSecret"
          placeholder-key="cluster.credential.openstack.auth.placeholders.appCredSecret"
          type="password"
          :mode="mode"
          @update:value="value.setData('applicationCredentialSecret', $event);"
        />
      </div> 
    </div>
    <BusyButton
      ref="connect"
      label-key="cluster.credential.openstack.auth.actions.authenticate"
      :disabled="step !== 1 || !canAuthenticate"
      class="mt-20"
      @clicked="connect"
    />

    <button
      class="btn role-primary mt-20 ml-20"
      :disabled="busy || step === 1"
      @click="clear"
    >
      {{ t('cluster.credential.openstack.auth.actions.edit') }}
    </button>

    <Banner
      v-if="error"
      class="mt-20"
      color="error"
    >
      {{ error }}
    </Banner>

    <Banner
      v-if="errorAllowHost"
      color="error"
      class="allow-list-error"
    >
      <div>
        {{ t('cluster.credential.openstack.auth.errors.notAllowed', { hostname }) }}
      </div>
      <button
        :disabled="allowBusy"
        class="btn ml-10 role-primary"
        @click="addHostToAllowList"
      >
        {{ t('cluster.credential.openstack.auth.actions.addToAllowList') }}
      </button>
    </Banner>
    <div
      v-if="projects"
      class="row mt-20"
    >
      <div class="col span-6">
        <LabeledSelect
          v-model:value="project"
          label-key="cluster.credential.openstack.auth.fields.project"
          :options="projectOptions"
          :searchable="false"
        />
      </div>
      <div
        v-if="regions"
        class="col span-6"
      >
        <LabeledSelect
          v-model:value="region"
          label-key="cluster.credential.openstack.auth.fields.region"
          :options="regionOptions"
          :searchable="false"
        />
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
  .allow-list-error {
    display: flex;

    > :first-child {
      flex: 1;
    }
  }
</style>
