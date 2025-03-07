<template>
	<div class="flex flex-shrink-0 space-x-2">
		<Dropdown
			placement="left"
			:button="{ icon: 'more-horizontal', appearance: 'minimal' }"
			:options="[
				!query.doc.is_stored && !query.doc.is_native_query
					? {
							label: 'Store Query',
							icon: 'bookmark',
							handler: storeQuery,
					  }
					: null,
				{
					label: 'Execute (⌘+E)',
					icon: 'play',
					handler: query.execute,
				},
				query.isOwner
					? {
							label: 'Share',
							icon: 'share-2',
							handler: () => (show_share_dialog = true),
					  }
					: null,
				{
					label: 'Pivot',
					icon: 'git-branch',
					handler: () => (show_pivot_dialog = true),
				},
				{
					label: 'Unpivot',
					icon: 'git-merge',
					handler: resetPivot,
				},
				{
					label: 'Reset',
					icon: 'refresh-ccw',
					handler: () => (show_reset_dialog = true),
				},
				!query.doc.is_native_query
					? {
							label: 'View SQL',
							icon: 'help-circle',
							handler: () => (show_sql_dialog = true),
					  }
					: null,
				{
					label: !query.doc.is_native_query ? 'Write SQL' : 'Build SQL',
					icon: 'codesandbox',
					handler: () => (show_convert_query_dialog = true),
				},
				{
					label: 'Duplicate',
					icon: 'copy',
					handler: duplicateQuery,
				},
				{
					label: 'Download CSV',
					icon: 'download',
					handler: downloadCSV,
				},
				{
					label: 'Delete',
					icon: 'trash-2',
					handler: () => (show_delete_dialog = true),
				},
			]"
		/>

		<Dialog
			:options="{
				title: 'Delete Query',
				icon: { name: 'trash', appearance: 'danger' },
			}"
			v-model="show_delete_dialog"
			:dismissable="true"
		>
			<template #body-content>
				<p class="text-base text-gray-600">Are you sure you want to delete this query?</p>
			</template>
			<template #actions>
				<Button
					appearance="danger"
					:loading="query.delete.loading"
					@click="
						() => {
							query.delete.submit().then(() => {
								builder?.closeQuery(query.name)
								show_delete_dialog = false
							})
						}
					"
				>
					Yes
				</Button>
			</template>
		</Dialog>

		<Dialog
			:options="{
				title: `Convert to ${!query.doc.is_native_query ? 'Native' : 'Builder'} Query`,
				icon: { name: 'info', appearance: 'warning' },
			}"
			v-model="show_convert_query_dialog"
			:dismissable="true"
		>
			<template #body-content>
				<p class="text-base text-gray-600">
					Are you sure you want to convert this query to a
					{{ !query.doc.is_native_query ? 'native' : 'builder' }}
					query? This will overwrite the existing query.
				</p>
			</template>
			<template #actions>
				<Button
					appearance="warning"
					:loading="query.convert.loading"
					@click="
						() => {
							query.convert.submit().then(() => {
								show_convert_query_dialog = false
							})
						}
					"
				>
					Yes
				</Button>
			</template>
		</Dialog>

		<Dialog
			:options="{
				title: 'Reset Query',
				icon: { name: 'alert-circle', appearance: 'danger' },
			}"
			v-model="show_reset_dialog"
			:dismissable="true"
		>
			<template #body-content>
				<p class="text-base text-gray-600">Are you sure you want to reset this query?</p>
			</template>
			<template #actions>
				<Button
					appearance="danger"
					:loading="query.reset.loading"
					@click="
						() => {
							query.reset.submit().then(() => {
								show_reset_dialog = false
							})
						}
					"
				>
					Yes
				</Button>
			</template>
		</Dialog>

		<Dialog
			:options="{ title: 'Generated SQL', size: '3xl' }"
			v-model="show_sql_dialog"
			:dismissable="true"
		>
			<template #body-content>
				<div class="relative">
					<p
						class="rounded-md border bg-gray-100 p-2 text-base text-gray-600"
						style="font-family: 'Fira Code'"
						v-html="formattedSQL"
					></p>
					<Button
						icon="copy"
						appearance="white"
						class="absolute bottom-2 right-2"
						@click="copySQL"
					></Button>
				</div>
			</template>
		</Dialog>

		<Dialog
			:options="{ title: 'Pivot Transform' }"
			v-model="show_pivot_dialog"
			:dismissable="true"
		>
			<template #body-content>
				<div class="space-y-4">
					<Input
						type="select"
						label="Pivot Column"
						v-model="pivot.column"
						:options="pivotOptions"
					/>
					<Input
						type="select"
						label="Index Column"
						v-model="pivot.index"
						:options="indexOptions"
					/>
					<Input
						type="select"
						label="Value Column"
						v-model="pivot.value"
						:options="valueOptions"
					/>
				</div>
			</template>
			<template #actions>
				<Button
					appearance="primary"
					@click="applyPivotTransform"
					:disabled="pivotDisabled"
					:loading="query.addTransform?.loading"
				>
					Apply
				</Button>
			</template>
		</Dialog>
	</div>

	<ShareDialog
		v-model:show="show_share_dialog"
		:resource-type="query.doc.doctype"
		:resource-name="query.doc.name"
	/>
</template>

<script setup>
import ShareDialog from '@/components/ShareDialog.vue'
import { useMagicKeys } from '@vueuse/core'
import { Dialog, Dropdown } from 'frappe-ui'
import { computed, inject, nextTick, reactive, ref, watch } from 'vue'
import { useRouter } from 'vue-router'

const props = defineProps(['query'])
const query = props.query || inject('query')
const builder = inject('queryBuilder')

const show_reset_dialog = ref(false)
const show_delete_dialog = ref(false)
const show_sql_dialog = ref(false)
const show_pivot_dialog = ref(false)
const show_share_dialog = ref(false)
const show_convert_query_dialog = ref(false)

const keys = useMagicKeys()
const cmdE = keys['Meta+E']
watch(cmdE, (value) => value && query.execute())

const pivot = reactive({
	column: null,
	index: null,
	value: null,
})

const formattedSQL = computed(() => {
	return query.doc.sql.replaceAll('\n', '<br>').replaceAll('      ', '&ensp;&ensp;&ensp;&ensp;')
})

const $router = useRouter()
const $notify = inject('$notify')
function duplicateQuery() {
	query.duplicate.submit().then(async (res) => {
		await nextTick()
		builder?.openQuery(res.message)
		$notify({
			appearance: 'success',
			title: 'Query Duplicated',
		})
	})
}

function storeQuery() {
	query.store.submit().then((res) => {
		$notify({
			appearance: 'success',
			title: 'Query Stored',
		})
	})
}

function copySQL() {
	if (navigator.clipboard) {
		navigator.clipboard.writeText(query.doc.sql)
		$notify({
			appearance: 'success',
			title: 'SQL Copied',
		})
	} else {
		$notify({
			appearance: 'warning',
			title: 'Copy to clipboard not supported',
		})
	}
}

const pivotOptions = computed(() =>
	[''].concat(
		query.columns.indexOptions
			.map((option) => option.label)
			.filter((option) => option !== pivot.index)
	)
)
const indexOptions = computed(() =>
	[''].concat(
		query.columns.indexOptions
			.map((option) => option.label)
			.filter((option) => option !== pivot.column)
	)
)
const valueOptions = computed(() =>
	[''].concat(query.columns.valueOptions.map((option) => option.label))
)

function applyPivotTransform() {
	query.addTransform
		.submit({
			type: 'Pivot',
			options: {
				column: pivot.column,
				index: pivot.index,
				value: pivot.value,
			},
		})
		.then(() => {
			show_pivot_dialog.value = false
			pivot.column = null
			pivot.index = null
			pivot.value = null
		})
}

function resetPivot() {
	query.resetTransforms.submit()
}

function downloadCSV() {
	let data = query.results.data
	if (data.length === 0) return
	data[0] = data[0].map((d) => d.split('::')[0])
	const csvString = data.map((row) => row.join(',')).join('\n')
	const blob = new Blob([csvString], { type: 'text/csv' })
	const url = window.URL.createObjectURL(blob)
	const a = document.createElement('a')
	a.setAttribute('hidden', '')
	a.setAttribute('href', url)
	a.setAttribute('download', `${query.doc.title || 'data'}.csv`)
	document.body.appendChild(a)
	a.click()
	document.body.removeChild(a)
}

const pivotDisabled = computed(() => {
	return !pivot.column || !pivot.index || !pivot.value
})
</script>
