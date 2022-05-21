<template>
	<div :class="$style.container">
		<LoadingView v-if="loading" />
		<div v-else id="app" :class="$style.container">		
			<div id="header" :class="$style.header" >
				<router-view name="header"></router-view>
			</div>
			<div v-if="!viewerMode" id="sidebar" :class="$style.sidebar" >
				<router-view name="sidebar"></router-view>
			</div>
			<div v-if="openForm" id="myModal" :class="$style.formmodal" >
			  <!-- Modal content -->
			  <div :class="$style.contentform">	
				<h2><p>Need a validation !</p></h2>
				<br/>
				<br/>
				<p v-for="item in dataForm.data" :key="item.display">
					<label for="item.input" v-if="item.input!=''"> {{ item.input }} :</label>
					<br/>

					<input v-model="item.input_value" placeholder="" v-if="item.input!=''">
					<br/>
					<span v-html="item.display">
					</span>
				</p>				
				<br/>
				<br/>
				<button :class="$style.buttonform" style="background-color: #FF0000;" v-on:click="sendResult('nok')">Reject</button>
				<button :class="$style.buttonform" style="background-color: #4CAF50;" v-on:click="sendResult('ok')">Accept</button>
			  </div>
			</div>	
			<div id="content" :class="$style.content">
				<router-view />
			</div>
			<Modals />
			<Telemetry />	
			
		</div>	
	</div>
	
</template> 

<script lang="ts">
import Modals from './components/Modals.vue';
import LoadingView from './views/LoadingView.vue';
import Telemetry from './components/Telemetry.vue';
import { HIRING_BANNER, VIEWS } from './constants';

import mixins from 'vue-typed-mixins';
import { showMessage } from './components/mixins/showMessage';
import { IUser } from './Interface';
import { mapGetters } from 'vuex';
import { userHelpers } from './components/mixins/userHelpers';
import { addHeaders, loadLanguage } from './plugins/i18n';
import { restApi } from '@/components/mixins/restApi';
import { io } from "socket.io-client";

export default mixins(
	showMessage,
	userHelpers,
	restApi,
).extend({
	name: 'App',
	components: {
		LoadingView,
		Telemetry,
		Modals,
	},
	computed: {
		...mapGetters('settings', ['isHiringBannerEnabled', 'isTemplatesEnabled', 'isTemplatesEndpointReachable', 'isUserManagementEnabled', 'showSetupPage']),
		...mapGetters('users', ['currentUser']),
		defaultLocale (): string {
			return this.$store.getters.defaultLocale;
		},
	},
	data() {
		return {
			loading: true,
			openForm: false,
			socket: io(""),
			dataFormRes:{"status":"","data":[]},
			dataForm: {"data":[]},
			pathNameWorkflow: "/workflow/",
			prefixTopicWsToListen: "/cmd-",
			currentWorkflowId: "",
			viewerMode:false,
		};
	},
	methods: {
		async initSettings(): Promise<void> {
			try {
				await this.$store.dispatch('settings/getSettings');
			} catch (e) {
				this.$showToast({
					title: this.$locale.baseText('startupError'),
					message: this.$locale.baseText('startupError.message'),
					type: 'error',
					duration: 0,
				});

				throw e;
			}
		},
		async loginWithCookie(): Promise<void> {
			try {
				await this.$store.dispatch('users/loginWithCookie');
			} catch (e) {}
		},
		async initTemplates(): Promise<void> {
			if (!this.isTemplatesEnabled) {
				return;
			}

			try {
				await this.$store.dispatch('settings/testTemplatesEndpoint');
			} catch (e) {
			}
		},
		logHiringBanner() {
			if (this.isHiringBannerEnabled && this.$route.name !== VIEWS.DEMO) {
				console.log(HIRING_BANNER); // eslint-disable-line no-console
			}
		},
		async initialize(): Promise<void> {
			await this.initSettings();
			await Promise.all([this.loginWithCookie(), this.initTemplates()]);
		},
		trackPage() {
			this.$store.commit('ui/setCurrentView', this.$route.name);
			if (this.$route && this.$route.meta && this.$route.meta.templatesEnabled) {
				this.$store.commit('templates/setSessionId');
			}
			else {
				this.$store.commit('templates/resetSessionId'); // reset telemetry session id when user leaves template pages
			}

			this.$telemetry.page(this.$route);
		},
		authenticate() {
			// redirect to setup page. user should be redirected to this only once
			if (this.isUserManagementEnabled && this.showSetupPage) {
				if (this.$route.name === VIEWS.SETUP) {
					return;
				}

				this.$router.replace({ name: VIEWS.SETUP });
				return;
			}

			if (this.canUserAccessCurrentRoute()) {
				return;
			}

			// if cannot access page and not logged in, ask to sign in
			const user = this.currentUser as IUser | null;
			if (!user) {
				const redirect =
					this.$route.query.redirect ||
					encodeURIComponent(`${window.location.pathname}${window.location.search}`);
				this.$router.replace({ name: VIEWS.SIGNIN, query: { redirect } });
				return;
			}

			// if cannot access page and is logged in, respect signin redirect
			if (this.$route.name === VIEWS.SIGNIN && typeof this.$route.query.redirect === 'string') {
				const redirect = decodeURIComponent(this.$route.query.redirect);
				if (redirect.startsWith('/')) { // protect against phishing
					this.$router.replace(redirect);
					return;
				}
			}

			// if cannot access page and is logged in
			this.$router.replace({ name: VIEWS.HOMEPAGE });
		},
		redirectIfNecessary() {
			const redirect = this.$route.meta && typeof this.$route.meta.getRedirect === 'function' && this.$route.meta.getRedirect(this.$store);
			if (redirect) {
				this.$router.replace(redirect);
			}
		},
		
		sendResult(ack: string){
			if(this.socket)
			{
				this.dataFormRes.status=ack;
				this.dataFormRes.data=this.dataForm.data;
				/* eslint-disable no-console */
				//console.log(this.dataForm);
				this.socket.emit(this.prefixTopicWsToListen+this.currentWorkflowId+"/"+this.$store.getters.sessionId+"/ack",this.dataForm);
			}			
			this.openForm=false;
		},
	},
	async mounted() {
		await this.initialize();

		if(process.env.VUE_APP_VIEWER_MODE === 'true'){
			this.viewerMode=true;
		}

		this.logHiringBanner();
		this.authenticate();
		this.redirectIfNecessary();
		
		this.socket = io(window.location.hostname+":42542");
	
		/* eslint-disable no-console */
		if(this.socket)
		{		
			console.log("SocketIo Client Connected to server"); // true
	
			this.socket.on("connect", () => {				
				this.socket.onAny((eventName, ...args) => {	
					this.currentWorkflowId=window.location.pathname.replace(this.pathNameWorkflow,"");
					if(eventName===this.prefixTopicWsToListen+this.currentWorkflowId+"/"+this.$store.getters.sessionId) {						
						this.openForm=true;			
						this.dataForm=JSON.parse(args[0]);
					}
				});
			});

			this.socket.on("disconnect", () => {
				console.log(this.socket.connected); // false
			});
		}

		this.loading = false;
		this.openForm = false;

		this.trackPage();
		this.$externalHooks().run('app.mount');

		if (this.defaultLocale !== 'en') {
			const headers = await this.restApi().getNodeTranslationHeaders();
			if (headers) addHeaders(headers, this.defaultLocale);
		}
	},
	watch: {
		$route(route) {
			this.authenticate();
			this.redirectIfNecessary();

			this.trackPage();
		},
		defaultLocale(newLocale) {
			loadLanguage(newLocale);
		},
	},
});
</script>

<style lang="scss" module>
.container {
	height: 100%;
	width: 100%;
}

.content {
	composes: container;
	background-color: var(--color-background-light);
	position: relative;
}

.header {
	z-index: 10;
	position: fixed;
	width: 100%;
}
	
.sidebar {
	z-index: 15;
	position: fixed;
}

/* The Modal (background) */
.formmodal {
  position: absolute;
  top: 20%;
  left: 30%;
  z-index: 10;
}

/* Modal Content/Box */
.contentform {
  position: flex;
  background-color: #fefefe;
  margin: 0% auto; /* 15% from the top and centered */
  padding: 100px;
  border: 1px solid #888;
  width: 99%; /* Could be more or less, depending on screen size */
}

.buttonform {
    border: none;
    color: white;
    padding: 15px 32px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    margin: 4px 2px;
    cursor: pointer;
	border-radius: 8px;
    width: calc(50% - 4px);
}

</style>

