<template>
	<AppContent>
		<AppDetailsToggle v-if="showMessage" @close="hideMessage" />
		<div id="app-content-wrapper">
			<AppContentList
				v-infinite-scroll="onScroll"
				v-shortkey.once="shortkeys"
				infinite-scroll-immediate-check="false"
				:show-details="showMessage"
				:infinite-scroll-disabled="false"
				:infinite-scroll-distance="10"
				@shortkey.native="onShortcut">
				<Mailbox
					v-if="!mailbox.isPriorityInbox"
					:account="account"
					:mailbox="mailbox"
					:search-query="query"
					:bus="bus" />
				<template v-else>
					<div class="app-content-list-item">
						<SectionTitle class="important" :name="t('mail', 'Important')" />
						<Popover trigger="hover focus">
							<button slot="trigger" :aria-label="t('mail', 'Important info')" class="button icon-info" />
							{{ importantInfo }}
						</Popover>
					</div>
					<Mailbox
						class="nameimportant"
						:account="unifiedAccount"
						:mailbox="unifiedInbox"
						:search-query="appendToSearch('is:important')"
						:paginate="'manual'"
						:is-priority-inbox="true"
						:initial-page-size="5"
						:collapsible="true"
						:bus="bus" />
					<SectionTitle class="app-content-list-item starred" :name="t('mail', 'Favorites')" />
					<Mailbox
						class="namestarred"
						:account="unifiedAccount"
						:mailbox="unifiedInbox"
						:search-query="appendToSearch('is:starred not:important')"
						:paginate="'manual'"
						:is-priority-inbox="true"
						:initial-page-size="5"
						:bus="bus" />
					<SectionTitle class="app-content-list-item other" :name="t('mail', 'Other')" />
					<Mailbox
						class="nameother"
						:account="unifiedAccount"
						:mailbox="unifiedInbox"
						:open-first="false"
						:search-query="appendToSearch('not:starred not:important')"
						:is-priority-inbox="true"
						:bus="bus" />
				</template>
			</AppContentList>
			<NewMessageDetail v-if="newMessage" />
			<Message v-else-if="showMessage" @delete="deleteMessage" />
			<NoMessageSelected v-else-if="hasMessages && !isMobile" />
		</div>
	</AppContent>
</template>

<script>
import AppContent from '@nextcloud/vue/dist/Components/AppContent'
import AppContentList from '@nextcloud/vue/dist/Components/AppContentList'
import Popover from '@nextcloud/vue/dist/Components/Popover'
import infiniteScroll from 'vue-infinite-scroll'
import isMobile from '@nextcloud/vue/dist/Mixins/isMobile'
import SectionTitle from './SectionTitle'
import Vue from 'vue'

import AppDetailsToggle from './AppDetailsToggle'
import logger from '../logger'
import Mailbox from './Mailbox'
import Message from './Message'
import NewMessageDetail from './NewMessageDetail'
import NoMessageSelected from './NoMessageSelected'
import { normalizedEnvelopeListId } from '../store/normalization'
import { UNIFIED_ACCOUNT_ID, UNIFIED_INBOX_ID } from '../store/constants'

export default {
	name: 'MailboxMessage',
	directives: {
		infiniteScroll,
	},
	components: {
		AppContent,
		Popover,
		AppContentList,
		AppDetailsToggle,
		Mailbox,
		Message,
		NewMessageDetail,
		NoMessageSelected,
		SectionTitle,
	},
	mixins: [isMobile],
	props: {
		account: {
			type: Object,
			required: true,
		},
		mailbox: {
			type: Object,
			required: true,
		},
	},
	data() {
		return {
			alive: false,
			// eslint-disable-next-line
			importantInfo: t('mail', 'Messages will automatically be marked as important based on which messages you interacted with or marked as important. In the beginning you might have to manually change the importance to teach the system, but it will improve over time.'),
			bus: new Vue(),
			searchQuery: undefined,
			shortkeys: {
				del: ['del'],
				flag: ['s'],
				next: ['arrowright'],
				prev: ['arrowleft'],
				refresh: ['r'],
				unseen: ['u'],
			},
		}
	},
	computed: {
		unifiedAccount() {
			return this.$store.getters.getAccount(UNIFIED_ACCOUNT_ID)
		},
		unifiedInbox() {
			return this.$store.getters.getMailbox(UNIFIED_INBOX_ID)
		},
		hasMessages() {
			// it actually should be `return this.$store.getters.getEnvelopes(this.account.id, this.mailbox.databaseId).length > 0`
			// but for some reason Vue doesn't track the dependencies on reactive data then and messages in submailboxes can't
			// be opened then
			const list = this.mailbox.envelopeLists[normalizedEnvelopeListId(this.searchQuery)]

			if (list === undefined) {
				return false
			}
			return list.length > 0
		},
		showMessage() {
			return (this.mailbox.isPriorityInbox === true || this.hasMessages) && this.$route.name === 'message'
		},
		query() {
			if (this.$route.params.filter === 'starred') {
				if (this.searchQuery) {
					return this.searchQuery + ' is:starred'
				}
				return 'is:starred'
			}
			return this.searchQuery
		},
		newMessage() {
			return (
				this.$route.params.threadId === 'new'
				|| this.$route.params.threadId === 'reply'
				|| this.$route.params.threadId === 'replyAll'
			)
		},
	},
	created() {
		this.alive = true

		window.addEventListener('DOMContentLoaded', (event) => {
			// eslint-disable-next-line no-new
			new OCA.Search(this.searchProxy, this.clearSearchProxy)
		})
	},
	beforeDestroy() {
		this.alive = false
	},
	methods: {
		hideMessage() {
			this.$router.replace({
				name: 'mailbox',
				params: {
					mailboxId: this.mailbox.databaseId,
					filter: this.$route.params.filter ? this.$route.params.filter : undefined,
				},
			})
		},
		deleteMessage(id) {
			this.bus.$emit('delete', id)
		},
		onScroll(event) {
			logger.debug('scroll', { event })

			this.bus.$emit('loadMore')
		},
		onShortcut(e) {
			this.bus.$emit('shortcut', e)
		},
		appendToSearch(str) {
			if (this.searchQuery === undefined) {
				return str
			}
			return this.searchQuery + ' ' + str
		},
		searchProxy(query) {
			if (this.alive) {
				this.search(query)
			}
		},
		clearSearchProxy() {
			if (this.alive) {
				this.clearSearch()
			}
		},
		search(query) {
			this.searchQuery = query
		},
		clearSearch() {
			this.searchQuery = undefined
		},
	},
}
</script>

<style lang="scss" scoped>
.v-popover > .trigger > {
	z-index: 1;
}
.icon-info {
	background-image: var(--icon-info-000);
}
.app-content-list-item:hover {
	background: transparent;
}
.button {
	background-color: var(--color-main-background);
	width: 44px;
	height: 44px;
	border: 0;
	margin-bottom: 17px;

	&:hover,
	&:focus {
		background-color: var(--color-background-dark);
	}
}
</style>
