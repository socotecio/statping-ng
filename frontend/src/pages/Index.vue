<template>
    <div class="container col-md-7 col-sm-12 sm-container mainPage">

      <Header/>

      <div v-if="errorState" class="row mt-5 mb-5">
        <div class="col-12 text-center mt-3 mb-3">
          <span class="text-danger font-weight-bold">⚠️ {{ errorState }}</span>
        </div>
      </div>

      <div v-else-if="loadingGroups || loadingServices || loadingMessages" class="row mt-5 mb-5">
        <div class="col-12 mt-5 mb-2 text-center">
          <font-awesome-icon icon="circle-notch" class="text-dim" size="2x" spin/>
        </div>
        <div class="col-12 text-center mt-3 mb-3">
          <span class="text-dim">{{ loadingMessage }}</span>
        </div>
      </div>

      <div v-else-if="groups.length === 0 && services.length === 0 && messages === null" class="row mt-5 mb-5">
          <div class="col-12 text-center mt-3 mb-3">
          <span class="text-dim">No group, service or message to display.</span>
          </div>
      </div>

      <div v-else>
          <div class="col-12 full-col-12">
              <MessageBlock v-for="message in messages" v-bind:key="message.id" :message="message" />
          </div>

          <div class="col-12 full-col-12" v-if="services_no_group && services_no_group.length">
              <div v-for="service in services_no_group" v-bind:key="service.id" class="list-group online_list mb-4">
                  <div class="list-group-item list-group-item-action">
                      <router-link class="no-decoration font-3" :to="serviceLink(service)">
                          {{service.name}}
                          <MessagesIcon :messages="service.messages || {}"/>
                      </router-link>
                      <span class="badge float-right" :class="serviceBadgeClass(service)">{{service.online ? $t('online') : $t('offline')}}</span>
                      <GroupServiceFailures :service="service"/>
                      <IncidentsBlock :service="service || {}"/>
                  </div>
              </div>
          </div>

          <Group v-for="group in groups" v-bind:key="group.id" :group="group" />

          <!-- <div class="col-12 full-col-12">
              <div v-for="service in services" :ref="service.id" v-bind:key="service.id">
                  <ServiceBlock :service="service" />
              </div>
          </div> -->
      </div>
  </div>
</template>

<script>
import Api from "@/API";

const Group = () => import(/* webpackChunkName: "index" */ '@/components/Index/Group')
const Header = () => import(/* webpackChunkName: "index" */ '@/components/Index/Header')
const MessageBlock = () => import(/* webpackChunkName: "index" */ '@/components/Index/MessageBlock')
const ServiceBlock = () => import(/* webpackChunkName: "index" */ '@/components/Service/ServiceBlock')
const GroupServiceFailures = () => import(/* webpackChunkName: "index" */ '@/components/Index/GroupServiceFailures')
const IncidentsBlock = () => import(/* webpackChunkName: "index" */ '@/components/Index/IncidentsBlock')
const MessagesIcon = () => import(/* webpackChunkName: "index" */ '@/components/Index/MessagesIcon')

export default {
    name: 'Index',
    components: {
      IncidentsBlock,
      GroupServiceFailures,
      ServiceBlock,
      MessageBlock,
      MessagesIcon,
      Group,
      Header
    },
    data() {
        return {
            loadingGroups: true,
            loadingServices: true,
            loadingMessages: true,
            loadingCore: true,
            messages: null,
            errorState: null
        };
    },
    computed: {
        loadingMessage() {
          if (this.loadingGroups) {
            return "Loading Groups";
          } else if (this.loadingServices) {
            return "Loading Services";
          } else if (this.loadingMessages) {
            return "Loading Announcements";
          }
            return ""; // To avoid an error if no loading message is displayed
        },
        groups() {
            return this.$store.getters.groupsInOrder
        },
        services() {
            return this.$store.getters.servicesInOrder
        },
        services_no_group() {
            return this.$store.getters.servicesNoGroup
        },
        core() {
            return this.$store.getters.core
        },
        oauth() {
          return this.$store.getters.oauth
        }
    },
    async mounted() {
      try {
        const isLoggedIn = await this.checkLogin();

        if (!isLoggedIn) {
          this.$cookies.remove("statping_auth");

          // Load Core + OAuth
          const core = await Api.core();
          const oauth = await Api.oauth();

          console.log("Core data:", core);
          console.log("OAuth data:", oauth);

          if (!core || !oauth) {
            this.errorState = "Failed to load core or oauth configuration.";
            return;
          }

          this.$store.commit('setCore', core);
          this.$store.commit('setOauth', oauth);

          // Keyckoak redirection
          if (oauth.keycloak_client_id) {
            const redirectUri = encodeURIComponent(core.domain + "/oauth/keycloak");
            window.location.href = `${oauth.keycloak_endpoint_auth}?client_id=${oauth.keycloak_client_id}&redirect_uri=${redirectUri}&response_type=code${this.keycloak_scopes(oauth)}`;
            return;
          }

          return;
        }
        await this.loadAppData();

      } catch (e) {
        console.error("Error in mounted hook:", e);
        this.errorState = "Unexpected error occurred during initialization.";
        this.loadingGroups = false;
        this.loadingServices = false;
        this.loadingMessages = false;
      }
    },
      methods: {
          async loadAppData() {
            try {
              await this.$store.dispatch('loadGroups');
            } catch (error) {
              console.error("Error loading groups :", error);
              this.errorState = "An error occurred while loading groups.";
            } finally {
              this.loadingGroups = false;
            }

            try {
              await this.$store.dispatch('loadServices');
            } catch (error) {
              console.error("Error loading services :", error);
              this.errorState = "An error occurred while loading services.";
            } finally {
              this.loadingServices = false;
            }

            try {
              await this.$store.dispatch('loadMessages');
              this.messages = this.$store.getters.messages?.filter(m => this.inRange(m) && m.service === 0) ?? null;
            } catch (error) {
              console.error("Error loading messages :", error);
              this.errorState = "An error occurred while loading messages.";
            } finally {
              this.loadingMessages = false;
            }
          },

        async checkLogin() {
          const token = this.$cookies.get('statping_auth')
          if (!token) {
            this.$store.commit('setLoggedIn', false);
            return false
          }
          try {
            const jwt = await Api.check_token(token)
            this.$store.commit('setAdmin', jwt.admin);
            if (jwt.username) {
              this.$store.commit('setLoggedIn', true);
            }
            return true
          } catch (e) {
            console.error("Error during login check:", e);
            this.errorState = "An error occurred during the login process."
            return false;
          }
        },
        serviceLink(service) {
            return `/services/${service.id}`
        },
        inRange(message) {
            return this.isBetween(this.now(), message.start_on, message.start_on === message.end_on ? this.maxDate().toISOString() : message.end_on)
        },
        now() {
            return new Date();
        },
        maxDate() {
            return new Date(8640000000000000);
        },
        isBetween(value, min, max) {
            return value >= new Date(min) && value <= new Date(max);
        },
        keycloak_scopes(oauth) {
          let scopes = [];

          // Add openid scope if needed
          if (oauth.keycloak_is_open_id && !scopes.includes("openid")) {
              scopes.push("openid");
          }

          // Add other scopes
          if (oauth.keycloak_scopes) {
            oauth.keycloak_scopes.split(",").forEach(scope => {
              const trimmedScope = scope.trim();
              if (trimmedScope && !scopes.includes(trimmedScope)) {
                scopes.push(trimmedScope);
              }
            });
          }

          // Return the scopes as a query string
          return scopes.length > 0 ? `&scope=${scopes.join(" ")}` : "";
        },
    }
}
</script>
