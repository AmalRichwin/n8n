<template>
	<div
		:class="['executions-sidebar', $style.container]"
		ref="container"
		data-test-id="executions-sidebar"
	>
		<div :class="$style.heading">
			<n8n-heading tag="h2" size="medium" color="text-dark">
				{{ $locale.baseText('generic.executions') }}
			</n8n-heading>
		</div>
		<div :class="$style.controls">
			<el-checkbox
				v-model="autoRefresh"
				@change="onAutoRefreshToggle"
				data-test-id="auto-refresh-checkbox"
			>
				{{ $locale.baseText('executionsList.autoRefresh') }}
			</el-checkbox>
			<n8n-popover trigger="click">
				<template #reference>
					<div :class="$style.filterButton">
						<n8n-button
							icon="filter"
							type="tertiary"
							size="medium"
							:active="statusFilterApplied"
							data-test-id="executions-filter-button"
						>
							<n8n-badge v-if="statusFilterApplied" theme="primary" class="mr-4xs">1</n8n-badge>
							{{ $locale.baseText('executionsList.filters') }}
						</n8n-button>
					</div>
				</template>
				<div :class="$style['filters-dropdown']">
					<div class="mb-s">
						<n8n-input-label
							:label="$locale.baseText('executions.ExecutionStatus')"
							:bold="false"
							size="small"
							color="text-base"
							class="mb-3xs"
						/>
						<n8n-select
							v-model="filter.status"
							size="small"
							ref="typeInput"
							:class="$style['type-input']"
							:placeholder="$locale.baseText('generic.any')"
							data-test-id="execution-status-select"
							@change="onFilterChange"
						>
							<n8n-option
								v-for="item in executionStatuses"
								:key="item.id"
								:label="item.name"
								:value="item.id"
								:data-test-id="`execution-status-${item.id}`"
							>
							</n8n-option>
						</n8n-select>
					</div>
					<div :class="[$style.filterMessage, 'mt-s']" v-if="statusFilterApplied">
						<n8n-link @click="resetFilters">
							{{ $locale.baseText('generic.reset') }}
						</n8n-link>
					</div>
				</div>
			</n8n-popover>
		</div>
		<div v-show="statusFilterApplied" class="mb-xs">
			<n8n-info-tip :bold="false">
				{{ $locale.baseText('generic.filtersApplied') }}
				<n8n-link @click="resetFilters" size="small">
					{{ $locale.baseText('generic.resetAllFilters') }}
				</n8n-link>
			</n8n-info-tip>
		</div>
		<div
			:class="$style.executionList"
			ref="executionList"
			data-test-id="current-executions-list"
			@scroll="loadMore(20)"
		>
			<div v-if="loading" class="mr-m">
				<n8n-loading :class="$style.loader" variant="p" :rows="1" />
				<n8n-loading :class="$style.loader" variant="p" :rows="1" />
				<n8n-loading :class="$style.loader" variant="p" :rows="1" />
			</div>
			<div v-if="executions.length === 0 && statusFilterApplied" :class="$style.noResultsContainer">
				<n8n-text color="text-base" size="medium" align="center">
					{{ $locale.baseText('executionsLandingPage.noResults') }}
				</n8n-text>
			</div>
			<execution-card
				v-else
				v-for="execution in executions"
				:key="execution.id"
				:execution="execution"
				:ref="`execution-${execution.id}`"
				:data-test-id="`execution-details-${execution.id}`"
				@refresh="onRefresh"
				@retryExecution="onRetryExecution"
			/>
			<div v-if="loadingMore" class="mr-m">
				<n8n-loading :class="$style.loader" variant="p" :rows="1" />
			</div>
		</div>
		<div :class="$style.infoAccordion">
			<executions-info-accordion :initiallyExpanded="false" />
		</div>
	</div>
</template>

<script lang="ts">
import ExecutionCard from '@/components/ExecutionsView/ExecutionCard.vue';
import ExecutionsInfoAccordion from '@/components/ExecutionsView/ExecutionsInfoAccordion.vue';
import { VIEWS } from '../../constants';
import { range as _range } from 'lodash';
import { IExecutionsSummary } from '@/Interface';
import { Route } from 'vue-router';
import Vue from 'vue';
import { PropType } from 'vue';
import { mapStores } from 'pinia';
import { useUIStore } from '@/stores/ui';
import { useWorkflowsStore } from '@/stores/workflows';

export default Vue.extend({
	name: 'executions-sidebar',
	components: {
		ExecutionCard,
		ExecutionsInfoAccordion,
	},
	props: {
		executions: {
			type: Array as PropType<IExecutionsSummary[]>,
			required: true,
		},
		loading: {
			type: Boolean,
			default: true,
		},
		loadingMore: {
			type: Boolean,
			default: false,
		},
	},
	data() {
		return {
			VIEWS,
			filter: {
				status: '',
			},
			autoRefresh: false,
			autoRefreshInterval: undefined as undefined | NodeJS.Timer,
		};
	},
	computed: {
		...mapStores(useUIStore, useWorkflowsStore),
		statusFilterApplied(): boolean {
			return this.filter.status !== '';
		},
		executionStatuses(): Array<{ id: string; name: string }> {
			return [
				{ id: 'crashed', name: this.$locale.baseText('executionsList.error') },
				{ id: 'new', name: this.$locale.baseText('executionsList.new') },
				{ id: 'error', name: this.$locale.baseText('executionsList.error') },
				{ id: 'running', name: this.$locale.baseText('executionsList.running') },
				{ id: 'success', name: this.$locale.baseText('executionsList.success') },
				{ id: 'waiting', name: this.$locale.baseText('executionsList.waiting') },
			];
		},
	},
	watch: {
		$route(to: Route, from: Route) {
			if (from.name === VIEWS.EXECUTION_PREVIEW && to.name === VIEWS.EXECUTION_HOME) {
				// Skip parent route when navigating through executions with back button
				this.$router.go(-1);
			}
		},
		'workflowsStore.activeWorkflowExecution'() {
			this.checkListSize();
			this.scrollToActiveCard();
		},
	},
	mounted() {
		this.autoRefresh = this.uiStore.executionSidebarAutoRefresh === true;
		if (this.autoRefresh) {
			this.autoRefreshInterval = setInterval(() => this.onRefresh(), 4000);
		}
		// On larger screens, we need to load more then first page of executions
		// for the scroll bar to appear and infinite scrolling is enabled
		this.checkListSize();
		this.scrollToActiveCard();
	},
	beforeDestroy() {
		if (this.autoRefreshInterval) {
			clearInterval(this.autoRefreshInterval);
			this.autoRefreshInterval = undefined;
		}
	},
	methods: {
		loadMore(limit = 20): void {
			if (!this.loading) {
				const executionsList = this.$refs.executionList as HTMLElement;
				if (executionsList) {
					const diff =
						executionsList.offsetHeight - (executionsList.scrollHeight - executionsList.scrollTop);
					if (diff > -10 && diff < 10) {
						this.$emit('loadMore', limit);
					}
				}
			}
		},
		onRetryExecution(payload: Object) {
			this.$emit('retryExecution', payload);
		},
		onRefresh(): void {
			this.$emit('refresh');
		},
		onFilterChange(): void {
			this.$emit('filterUpdated', this.prepareFilter());
		},
		reloadExecutions(): void {
			this.$emit('reloadExecutions');
		},
		onAutoRefreshToggle(): void {
			this.uiStore.executionSidebarAutoRefresh = this.autoRefresh;
			if (this.autoRefreshInterval) {
				// Clear any previously existing intervals (if any - there shouldn't)
				clearInterval(this.autoRefreshInterval);
				this.autoRefreshInterval = undefined;
			}
			if (this.autoRefresh) {
				this.autoRefreshInterval = setInterval(() => this.onRefresh(), 4 * 1000); // refresh data every 4 secs
			}
		},
		async resetFilters(): Promise<void> {
			this.filter.status = '';
			this.$emit('filterUpdated', this.prepareFilter());
		},
		prepareFilter(): object {
			return {
				finished: this.filter.status !== 'running',
				status: this.filter.status,
			};
		},
		checkListSize(): void {
			const sidebarContainer = this.$refs.container as HTMLElement;
			const currentExecutionCard = this.$refs[
				`execution-${this.workflowsStore.activeWorkflowExecution?.id}`
			] as Vue[];

			// Find out how many execution card can fit into list
			// and load more if needed
			if (sidebarContainer && currentExecutionCard?.length) {
				const cardElement = currentExecutionCard[0].$el as HTMLElement;
				const listCapacity = Math.ceil(sidebarContainer.clientHeight / cardElement.clientHeight);

				if (listCapacity > this.executions.length) {
					this.$emit('loadMore', listCapacity - this.executions.length);
				}
			}
		},
		scrollToActiveCard(): void {
			const executionsList = this.$refs.executionList as HTMLElement;
			const currentExecutionCard = this.$refs[
				`execution-${this.workflowsStore.activeWorkflowExecution?.id}`
			] as Vue[];

			if (
				executionsList &&
				currentExecutionCard?.length &&
				this.workflowsStore.activeWorkflowExecution
			) {
				const cardElement = currentExecutionCard[0].$el as HTMLElement;
				const cardRect = cardElement.getBoundingClientRect();
				const LIST_HEADER_OFFSET = 200;
				if (cardRect.top > executionsList.offsetHeight) {
					executionsList.scrollTo({ top: cardRect.top - LIST_HEADER_OFFSET });
				}
			}
		},
	},
});
</script>

<style module lang="scss">
.container {
	flex: 310px 0 0;
	background-color: var(--color-background-xlight);
	border-right: var(--border-base);
	padding: var(--spacing-l) 0 var(--spacing-l) var(--spacing-l);
	z-index: 1;
	overflow: hidden;
}

.heading {
	display: flex;
	justify-content: space-between;
	align-items: baseline;
	padding-right: var(--spacing-l);
}

.controls {
	padding: var(--spacing-s) 0 var(--spacing-xs);
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding-right: var(--spacing-l);

	button {
		display: flex;
		align-items: center;
	}
}

.executionList {
	height: calc(100% - 10.5em);
	overflow: auto;
	margin-bottom: var(--spacing-m);
	background-color: var(--color-background-xlight) !important;

	// Scrolling fader
	&::before {
		position: absolute;
		display: block;
		width: 270px;
		height: 6px;
		background: linear-gradient(to bottom, rgba(251, 251, 251, 1) 0%, rgba(251, 251, 251, 0) 100%);
		z-index: 999;
	}

	// Lower first execution card so fader is not visible when not scrolled
	& > div:first-child {
		margin-top: 3px;
	}
}

.infoAccordion {
	position: absolute;
	bottom: 0;
	margin-left: calc(-1 * var(--spacing-l));
	border-top: var(--border-base);

	& > div {
		width: 309px;
		background-color: var(--color-background-light);
		margin-top: 0 !important;
	}
}

.noResultsContainer {
	width: 100%;
	margin-top: var(--spacing-2xl);
	text-align: center;
}
</style>
