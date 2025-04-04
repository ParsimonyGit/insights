<template>
	<div class="flex w-full flex-col overflow-hidden">
		<div class="flex flex-shrink-0 items-center bg-white pb-3 pt-1">
			<Button icon="chevron-left" class="mr-2" @click="$emit('close')"> </Button>
			<div class="text-sm tracking-wide text-gray-600">JOIN</div>
		</div>
		<div class="flex flex-1 flex-col space-y-3 overflow-y-scroll">
			<div class="flex flex-col space-y-3">
				<div class="space-y-1 text-sm text-gray-600">
					<div class="font-light">Left Table</div>
					<LinkIcon :link="getQueryLink(editTable.table)" :show="editTable.label">
						<Input v-model="editTable.label" disabled class="h-8 cursor-not-allowed" />
					</LinkIcon>
				</div>
				<div class="space-y-1 text-sm text-gray-600">
					<div class="font-light">Join Type</div>
					<Autocomplete
						v-model="join.type"
						:options="joinTypeOptions"
						placeholder="Select a type..."
					/>
				</div>
				<div class="space-y-1 text-sm text-gray-600">
					<div class="font-light">Right Table</div>
					<LinkIcon :link="getQueryLink(join.with.value)" :show="join.with.label">
						<Autocomplete
							v-model="join.with"
							:options="query.tables.joinOptions"
							placeholder="Select a table..."
						/>
					</LinkIcon>
				</div>
				<div class="text-sm text-gray-600">
					<div class="flex items-end space-x-1">
						<div class="flex-1">
							<div class="mb-1 font-light">Left Column</div>
							<Autocomplete
								v-model="join.condition.left"
								:options="leftColumnOptions"
								placeholder="Select a Column"
							/>
						</div>
						<span class="flex h-8 items-center px-1 text-lg"> = </span>
						<div class="flex-1">
							<div class="mb-1 font-light">Right Column</div>
							<Autocomplete
								v-model="join.condition.right"
								:options="rightColumnOptions"
								placeholder="Select a Column"
							/>
						</div>
					</div>
				</div>
			</div>
			<div class="flex justify-end space-x-2">
				<Button :disabled="!editTable.join" @click="clear_join"> Clear </Button>
				<Button
					appearance="primary"
					:disabled="!join.with || !join.condition || !join.type"
					@click="applyJoin"
				>
					Apply
				</Button>
			</div>
		</div>
	</div>
</template>

<script setup>
import { ref, inject, watch } from 'vue'
import Autocomplete from '@/components/Controls/Autocomplete.vue'
import LinkIcon from '@/components/Controls/LinkIcon.vue'
import { getQueryLink } from '@/utils'

const emits = defineEmits(['close'])
const props = defineProps({
	table: {
		type: Object,
		required: true,
	},
})

const query = inject('query')
const join = ref({ with: {}, condition: {}, type: { label: 'Left', value: 'left' } })
const editTable = ref({ ...props.table }) // table that is being edited

if (editTable.value.join) {
	join.value = editTable.value.join
}

const joinTypeOptions = ref([
	{ label: 'Inner Join', value: 'inner' },
	{ label: 'Left Join', value: 'left' },
	{ label: 'Full Outer Join', value: 'full' },
])

watch(
	() => join.value.with,
	(newVal) => newVal && setJoinCondition()
)

const leftColumnOptions = ref([])
const rightColumnOptions = ref([])
function setJoinCondition() {
	const leftTable = editTable.value.table
	const rightTable = join.value.with.value

	if (!leftTable || !rightTable) return

	query.fetchJoinOptions
		.submit({
			left_table: leftTable,
			right_table: rightTable,
		})
		.then(({ message }) => {
			setJoinConditionOptions(message)
			if (message.saved_links.length) {
				setSavedJoinCondition(message.saved_links[0])
			}
		})
}
function setJoinConditionOptions({ left_columns, right_columns }) {
	leftColumnOptions.value = left_columns.map((c) => ({
		label: c.label,
		value: c.column,
	}))
	rightColumnOptions.value = right_columns.map((c) => ({
		label: c.label,
		value: c.column,
	}))
}
function setSavedJoinCondition({ left, right }) {
	join.value.condition = {
		left: leftColumnOptions.value.find((col) => col.value === left),
		right: rightColumnOptions.value.find((col) => col.value === right),
	}
}
function applyJoin() {
	editTable.value.join = join.value
	query.updateTable.submit({ table: editTable.value })
	emits('close')
}
function clear_join() {
	editTable.value.join = ''
	query.updateTable.submit({ table: editTable.value })
	emits('close')
}
</script>
