<template>
	<div id="moodle_prefs" class="section">
		<h2>
			<a class="icon icon-moodle" />
			{{ t('integration_moodle', 'Moodle integration') }}
		</h2>
		<p v-if="!connected" class="settings-hint">
			{{ t('integration_moodle', 'Your password is not stored. It is just used once to get an access token which will be used to interact with your Moodle account.') }}
		</p>
		<div id="moodle-content">
			<div class="moodle-grid-form">
				<label for="moodle-url">
					<a class="icon icon-link" />
					{{ t('integration_moodle', 'Moodle instance address') }}
				</label>
				<input id="moodle-url"
					v-model="state.url"
					type="text"
					:disabled="connected === true"
					:placeholder="t('integration_moodle', 'Moodle instance URL')"
					@input="onInput">
				<label
					v-show="connected !== true"
					for="moodle-login">
					<a class="icon icon-user" />
					{{ t('integration_moodle', 'Moodle login') }}
				</label>
				<input
					v-show="connected !== true"
					id="moodle-login"
					v-model="login"
					type="text"
					:placeholder="t('integration_moodle', 'Your user name')"
					@keyup.enter="onValidate">
				<label
					v-show="connected !== true"
					for="moodle-password">
					<a class="icon icon-password" />
					{{ t('integration_moodle', 'Moodle password') }}
				</label>
				<input
					v-show="connected !== true"
					id="moodle-password"
					v-model="password"
					type="password"
					:placeholder="t('integration_moodle', 'Your password')"
					@keyup.enter="onValidate">
			</div>
			<button v-if="showConnect && !connected"
				:class="{ loading: connecting }"
				@click="onValidate">
				<span class="icon icon-external" />
				{{ t('integration_moodle', 'Connect to Moodle') }}
			</button>
			<div v-if="connected" class="moodle-grid-form">
				<label class="moodle-connected">
					<a class="icon icon-checkmark-color" />
					{{ t('integration_moodle', 'Connected as {user}', { user: state.user_name }) }}
				</label>
				<button id="moodle-rm-cred" @click="onLogoutClick">
					<span class="icon icon-close" />
					{{ t('integration_moodle', 'Disconnect from Moodle') }}
				</button>
				<span />
			</div>
			<br>
			<input
				id="check-ssl-moodle"
				type="checkbox"
				class="checkbox"
				:checked="state.check_ssl"
				@input="onCheckSslChange">
			<label for="check-ssl-moodle">{{ t('integration_moodle', 'Check SSL certificate') }}</label>
			<br>
			<div v-if="connected && !state.search_disabled" id="moodle-search-block">
				<input
					id="search-moodle-courses"
					type="checkbox"
					class="checkbox"
					:checked="state.search_courses_enabled"
					@input="onSearchCoursesChange">
				<label for="search-moodle-courses">{{ t('integration_moodle', 'Enable searching for courses') }}</label>
				<br><br>
				<input
					id="search-moodle-modules"
					type="checkbox"
					class="checkbox"
					:checked="state.search_modules_enabled"
					@input="onSearchModulesChange">
				<label for="search-moodle-modules">{{ t('integration_moodle', 'Enable searching for course modules') }}</label>
				<br><br>
				<input
					id="search-moodle-upcoming"
					type="checkbox"
					class="checkbox"
					:checked="state.search_upcoming_enabled"
					@input="onSearchUpcomingChange">
				<label for="search-moodle-upcoming">{{ t('integration_moodle', 'Enable searching for upcoming events') }}</label>
				<br><br>
				<p v-if="searchEnabled" class="settings-hint">
					<span class="icon icon-details" />
					{{ t('integration_moodle', 'Warning, everything you type in the search bar will be sent to your Moodle instance.') }}
				</p>
			</div>
		</div>
	</div>
</template>

<script>
import { loadState } from '@nextcloud/initial-state'
import { generateUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import { delay } from '../utils'
import { showSuccess, showError } from '@nextcloud/dialogs'
import '@nextcloud/dialogs/styles/toast.scss'

export default {
	name: 'PersonalSettings',

	components: {
	},

	props: [],

	data() {
		return {
			state: loadState('integration_moodle', 'user-config'),
			login: '',
			password: '',
			connecting: false,
		}
	},

	computed: {
		showConnect() {
			return this.login && this.login !== ''
				&& this.password && this.password !== ''
		},
		connected() {
			return this.state.url && this.state.url !== ''
				&& this.state.user_name && this.state.user_name !== ''
		},
		searchEnabled() {
			return this.state.search_courses_enabled || this.state.search_modules_enabled || this.state.search_upcoming_enabled
		},
	},

	methods: {
		onLogoutClick() {
			this.state.user_name = ''
			this.saveOptions({ user_name: '', token: '' })
		},
		onCheckSslChange(e) {
			this.state.check_ssl = e.target.checked
			this.saveOptions({ check_ssl: this.state.check_ssl ? '1' : '0' })
		},
		onSearchCoursesChange(e) {
			this.state.search_courses_enabled = e.target.checked
			this.saveOptions({ search_courses_enabled: this.state.search_courses_enabled ? '1' : '0' })
		},
		onSearchModulesChange(e) {
			this.state.search_modules_enabled = e.target.checked
			this.saveOptions({ search_modules_enabled: this.state.search_modules_enabled ? '1' : '0' })
		},
		onSearchUpcomingChange(e) {
			this.state.search_upcoming_enabled = e.target.checked
			this.saveOptions({ search_upcoming_enabled: this.state.search_upcoming_enabled ? '1' : '0' })
		},
		onInput() {
			delay(() => {
				if (!this.state.url.startsWith('https://') && !this.state.url.startsWith('http://')) {
					this.state.url = 'https://' + this.state.url.replace(/^[a-zA-Z]*:\/\//, '')
				}
				this.saveOptions({ url: this.state.url })
			}, 2000)()
		},
		saveOptions(values) {
			const req = {
				values,
			}
			const url = generateUrl('/apps/integration_moodle/config')
			axios.put(url, req)
				.then((response) => {
					showSuccess(t('integration_moodle', 'Moodle options saved'))
				})
				.catch((error) => {
					showError(
						t('integration_moodle', 'Failed to save Moodle options')
						+ ': ' + error.response?.request?.responseText
					)
				})
				.then(() => {
				})
		},
		onValidate() {
			this.connecting = true
			const req = {
				login: this.login,
				password: this.password,
			}
			const url = generateUrl('/apps/integration_moodle/get-token')
			axios.post(url, req)
				.then((response) => {
					this.state.user_name = response.data.user_name
					this.password = ''
					showSuccess(t('integration_moodle', 'Successfully connected to Moodle!'))
				})
				.catch((error) => {
					let errorText = ''
					if (error.response?.request?.responseText) {
						const jsonError = JSON.parse(error.response.request.responseText)
						errorText = ': ' + jsonError.error
					}
					showError(
						t('integration_moodle', 'Failed to authenticate to Moodle')
						+ errorText
					)
				})
				.then(() => {
					this.connecting = false
				})
		},
	},
}
</script>

<style scoped lang="scss">
#moodle-search-block {
	margin-top: 30px;
}

.moodle-grid-form label {
	line-height: 38px;
}

.moodle-grid-form input {
	width: 100%;
}

.moodle-grid-form {
	max-width: 600px;
	display: grid;
	grid-template: 1fr / 1fr 1fr;
	button .icon {
		margin-bottom: -1px;
	}
}

#moodle_prefs .icon {
	display: inline-block;
	width: 32px;
}

#moodle_prefs .grid-form .icon {
	margin-bottom: -3px;
}

.icon-moodle {
	background-image: url(./../../img/app-dark.svg);
	background-size: 23px 23px;
	height: 23px;
	margin-bottom: -4px;
}

body.theme--dark .icon-moodle {
	background-image: url(./../../img/app.svg);
}

#moodle-content {
	margin-left: 40px;
}

#moodle-search-block .icon {
	width: 22px;
}

</style>
