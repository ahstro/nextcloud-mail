<template>
	<AppContentDetails id="mail-message">
		<Loading v-if="loading" />
		<Error
			v-else-if="!message"
			:error="error && error.message ? error.message : t('mail', 'Not found')"
			:message="errorMessage"
			:data="error"
		>
		</Error>
		<template v-else>
			<div id="mail-message-header">
				<div id="mail-message-header-fields">
					<h2 :title="message.subject">{{ message.subject }}</h2>
					<p class="transparency">
						<AddressList :entries="message.from" />
						to
						<!-- TODO: translate -->
						<AddressList :entries="message.to" />
						<template v-if="message.cc.length">
							(cc
							<!-- TODO: translate -->
							<AddressList :entries="message.cc" /><!--
							-->)
						</template>
					</p>
				</div>
				<div id="mail-message-actions">
					<div
						:class="
							hasMultipleRecipients
								? 'icon-reply-all-white button primary'
								: 'icon-reply-white button primary'
						"
						@click="hasMultipleRecipients ? replyAll() : replyMessage()"
					>
						<span class="action-label">{{ t('mail', 'Reply') }}</span>
					</div>
					<Actions id="mail-message-actions-menu" class="app-content-list-item-menu" menu-align="right">
						<ActionButton v-if="hasMultipleRecipients" icon="icon-reply" @click="replyMessage">
							{{ t('mail', 'Reply to sender only') }}
						</ActionButton>
						<ActionButton icon="icon-forward" @click="forwardMessage">
							{{ t('mail', 'Forward') }}
						</ActionButton>
						<ActionButton icon="icon-mail" @click="onToggleSeen">
							{{ envelope.flags.unseen ? t('mail', 'Mark read') : t('mail', 'Mark unread') }}
						</ActionButton>
						<ActionButton icon="icon-delete" @click.prevent="onDelete">
							{{ t('mail', 'Delete') }}
						</ActionButton>
					</Actions>
				</div>
			</div>
			<div class="mail-message-body">
				<div v-if="message.itineraries.length > 0" class="message-itinerary">
					<Itinerary :entries="message.itineraries" :message-id="message.messageId" />
				</div>
				<MessageHTMLBody v-if="message.hasHtmlBody" :url="htmlUrl" />
				<MessagePlainTextBody v-else :body="message.body" :signature="message.signature" />
				<MessageAttachments :attachments="message.attachments" />
				<div id="reply-composer"></div>
			</div>
		</template>
	</AppContentDetails>
</template>

<script>
import Actions from '@nextcloud/vue/dist/Components/Actions'
import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import AppContentDetails from '@nextcloud/vue/dist/Components/AppContentDetails'
import {generateUrl} from '@nextcloud/router'

import AddressList from './AddressList'
import {buildRecipients as buildReplyRecipients, buildReplySubject} from '../ReplyBuilder'
import Error from './Error'
import {getRandomMessageErrorMessage} from '../util/ErrorMessageFactory'
import Itinerary from './Itinerary'
import MessageHTMLBody from './MessageHTMLBody'
import MessagePlainTextBody from './MessagePlainTextBody'
import Loading from './Loading'
import logger from '../logger'
import MessageAttachments from './MessageAttachments'

export default {
	name: 'Message',
	components: {
		ActionButton,
		Actions,
		AddressList,
		AppContentDetails,
		Error,
		Itinerary,
		Loading,
		MessageAttachments,
		MessageHTMLBody,
		MessagePlainTextBody,
	},
	data() {
		return {
			loading: true,
			message: undefined,
			envelope: undefined,
			errorMessage: '',
			error: undefined,
			replyRecipient: {},
			replySubject: '',
		}
	},
	computed: {
		htmlUrl() {
			return generateUrl('/apps/mail/api/accounts/{accountId}/folders/{folderId}/messages/{id}/html', {
				accountId: this.message.accountId,
				folderId: this.message.folderId,
				id: this.message.id,
			})
		},
		replyTo() {
			return {
				accountId: this.message.accountId,
				folderId: this.message.folderId,
				messageId: this.message.id,
			}
		},
		hasMultipleRecipients() {
			return this.replyRecipient.to.concat(this.replyRecipient.cc).length > 1
		},
	},
	watch: {
		$route(to, from) {
			this.fetchMessage()
		},
	},
	created() {
		this.fetchMessage()
	},
	methods: {
		async fetchMessage() {
			this.loading = true
			this.message = undefined
			this.errorMessage = ''
			this.error = undefined
			this.replyRecipient = {}
			this.replySubject = ''

			const messageUid = this.$route.params.messageUid

			try {
				const [envelope, message] = await Promise.all([
					this.$store.dispatch('fetchEnvelope', messageUid),
					this.$store.dispatch('fetchMessage', messageUid),
				])
				logger.debug('envelope and message fetched', {envelope, message})
				// TODO: add timeout so that message isn't flagged when only viewed
				// for a few seconds
				if (message && message.uid !== this.$route.params.messageUid) {
					logger.debug("User navigated away, loaded message won't be shown nor flagged as seen")
					return
				}

				this.envelope = envelope
				this.message = message

				if (envelope === undefined || message === undefined) {
					logger.info('message could not be found', {messageUid})
					this.errorMessage = getRandomMessageErrorMessage()
					this.loading = false
					return
				}

				const account = this.$store.getters.getAccount(message.accountId)
				this.replyRecipient = buildReplyRecipients(message, {
					label: account.name,
					email: account.emailAddress,
				})

				this.replySubject = buildReplySubject(message.subject)

				this.loading = false

				if (envelope.flags.unseen) {
					return this.$store.dispatch('toggleEnvelopeSeen', envelope)
				}
			} catch (error) {
				logger.error('could not load message ', {messageUid, error})
				if (error.isError) {
					this.errorMessage = t('mail', 'Could not load your message')
					this.error = error
					this.loading = false
				}
			}
		},
		replyMessage() {
			this.$router.push({
				name: 'message',
				params: {
					accountId: this.$route.params.accountId,
					folderId: this.$route.params.folderId,
					messageUid: 'reply',
				},
				query: {
					uid: this.message.uid,
				},
			})
		},
		replyAll() {
			this.$router.push({
				name: 'message',
				params: {
					accountId: this.$route.params.accountId,
					folderId: this.$route.params.folderId,
					messageUid: 'replyAll',
				},
				query: {
					uid: this.message.uid,
				},
			})
		},
		forwardMessage() {
			this.$router.push({
				name: 'message',
				params: {
					accountId: this.$route.params.accountId,
					folderId: this.$route.params.folderId,
					messageUid: 'new',
				},
				query: {
					uid: this.message.uid,
				},
			})
		},
		onToggleSeen() {
			this.$store.dispatch('toggleEnvelopeSeen', this.envelope)
		},
		onDelete() {
			this.$emit('delete', {
				envelope: this.envelope,
				message: this.message,
			})
			this.$store.dispatch('deleteMessage', this.message)
		},
	},
}
</script>

<style lang="scss">
#mail-message {
	flex-grow: 1;
}

.mail-message-body {
	margin-bottom: 100px;
}

#mail-message-header {
	display: flex;
	flex-direction: row;
	justify-content: space-between;
	align-items: center;
	padding: 30px 0;
	// somehow ios doesn't care about this !important rule
	// so we have to manually set left/right padding to chidren
	// for 100% to be used
	box-sizing: content-box !important;
	height: 44px;
	width: 100%;
}

#mail-message-header-fields {
	// initial width
	width: 0;
	padding-left: 38px;
	// grow and try to fill 100%
	flex: 1 1 auto;
	h2,
	p {
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		padding-bottom: 7px;
		margin-bottom: 0;
	}

	.transparency {
		opacity: 0.6;
		a {
			font-weight: bold;
		}
	}
}

#mail-content,
.mail-message-attachments {
	margin: 10px 38px 50px 38px;
}

.mail-message-attachments {
	margin-top: 10px;
}

#mail-content iframe {
	width: 100%;
}

#show-images-text {
	display: none;
}

#mail-content a,
.mail-signature a {
	color: #07d;
	border-bottom: 1px dotted #07d;
	text-decoration: none;
	word-wrap: break-word;
}

#mail-message-actions {
	display: flex;
	flex-direction: row;
	justify-content: flex-end;
	margin-left: 10px;
	margin-right: 22px;
	height: 44px;
}

.icon-reply-white,
.icon-reply-all-white {
	height: 44px;
	min-width: 44px;
	margin: 0;
	padding: 9px 18px 10px 32px;
}

/* Show action button label and move icon to the left
   on screens larger than 600px */
@media only screen and (max-width: 600px) {
	.action-label {
		display: none;
	}
}
@media only screen and (min-width: 600px) {
	.icon-reply-white,
	.icon-reply-all-white {
		background-position: 12px center;
	}
}

#mail-message-actions-menu {
	margin-left: 4px;
}

@media print {
	#mail-message-header-fields {
		position: relative;
	}

	#header,
	#app-navigation,
	#reply-composer,
	#forward-button,
	#mail-message-has-blocked-content,
	.app-content-list,
	.message-composer,
	.mail-message-attachments {
		display: none !important;
	}
	#app-content {
		margin-left: 0 !important;
	}
	.mail-message-body {
		margin-bottom: 0 !important;
	}
}
</style>
