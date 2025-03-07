<template>
	<div class="flex flex-col space-y-3">
		<div class="space-y-1 text-sm text-gray-600">
			<div class="font-light">Column</div>
			<Autocomplete
				v-model="filter.column"
				:options="columnOptions"
				placeholder="Select a column..."
			/>
		</div>
		<div class="space-y-1 text-sm text-gray-600">
			<div class="font-light">Operator</div>
			<Autocomplete
				v-model="filter.operator"
				:options="operatorOptions"
				placeholder="Select operator..."
			/>
		</div>
		<div class="space-y-1 text-sm text-gray-600">
			<div class="font-light">Value</div>
			<Autocomplete
				v-if="showValueOptions"
				v-model="filter.value"
				:options="valueOptions"
				:placeholder="valuePlaceholder"
				@inputChange="checkAndFetchColumnValues"
				:loading="query.fetchColumnValues.loading"
			/>
			<TimespanPicker
				v-else-if="showTimespanPicker"
				id="value"
				v-model="filter.value"
				:placeholder="valuePlaceholder"
			/>
			<ListPicker
				v-else-if="showListPicker"
				:value="filter.value.value"
				@change="
					(event) => {
						if (!filter.value.value) {
							filter.value = {
								value: null,
								label: null,
							}
						}
						filter.value.value = event
					}
				"
				:options="valueOptions"
				:placeholder="valuePlaceholder"
				@inputChange="checkAndFetchColumnValues"
				:loading="query.fetchColumnValues.loading"
			/>
			<DateRangePicker
				v-else-if="showDatePicker && filter.operator.value === 'between'"
				id="value"
				:value="filter.value.value"
				:placeholder="valuePlaceholder"
				:formatter="formatDate"
				@change="
					(date) => {
						filter.value = {
							value: date,
							label: formatDate(date),
						}
					}
				"
			/>
			<DatePicker
				v-else-if="showDatePicker"
				id="value"
				:value="filter.value.value"
				:placeholder="valuePlaceholder"
				:formatter="formatDate"
				@change="
					(date) => {
						filter.value = {
							value: date,
							label: formatDate(date),
						}
					}
				"
			/>
			<input
				v-else
				type="text"
				v-model="filter.value.value"
				:placeholder="valuePlaceholder"
				class="form-input block h-8 w-full select-none rounded-md placeholder-gray-500 placeholder:text-sm"
			/>
		</div>
		<div class="flex justify-end">
			<Button @click="apply" appearance="primary" :disabled="applyDisabled"> Apply </Button>
		</div>
	</div>
</template>

<script setup>
import Autocomplete from '@/components/Controls/Autocomplete.vue'
import DatePicker from '@/components/Controls/DatePicker.vue'
import DateRangePicker from '@/components/Controls/DateRangePicker.vue'
import ListPicker from '@/components/Controls/ListPicker.vue'
import TimespanPicker from '@/components/Controls/TimespanPicker.vue'

import { formatDate, isEmptyObj } from '@/utils'
import { debounce } from 'frappe-ui'
import { computed, inject, reactive, watch } from 'vue'

const query = inject('query')

const props = defineProps({
	filter: {
		type: Object,
		default: {
			column: {},
			operator: {},
			value: {},
		},
	},
})
const emit = defineEmits(['filter-select'])

const initalData = {
	column: {},
	operator: {},
	value: {},
}
let filter = reactive(initalData)
if (props.filter && props.filter.type) {
	const simpleFilter = query.filters.convertIntoSimpleFilter(props.filter)
	filter = reactive(simpleFilter)
}

const columnOptions = query.columns.options
const operatorOptions = computed(() => query.columns.getOperatorOptions(filter.column?.type))

const showDatePicker = computed(() => {
	return (
		['Date', 'Datetime'].includes(filter.column?.type) &&
		['=', '!=', '>', '>=', '<', '<=', 'between'].includes(filter.operator?.value)
	)
})
const showTimespanPicker = computed(
	() =>
		['Date', 'Datetime'].includes(filter.column?.type) && filter.operator?.value === 'timespan'
)

const showListPicker = computed(
	() => ['in', 'not_in'].includes(filter.operator?.value) && filter.column?.type == 'String'
)

const showValueOptions = computed(
	() => ['=', '!=', 'is'].includes(filter.operator?.value) && filter.column?.type == 'String'
)

const valueOptions = computed(() => {
	if (filter.operator?.value == 'is') {
		return [
			{ label: 'Set', value: 'set' },
			{ label: 'Not Set', value: 'not set' },
		]
	}
	return query.fetchColumnValues.data?.message
})

const valuePlaceholder = computed(() => {
	if (showDatePicker.value) {
		return 'Select a date...'
	}
	if (isEmptyObj(filter.operator)) {
		return 'Type a value...'
	}
	if (filter.operator?.value == 'between') {
		return 'Type two comma separated values...'
	}
	if (filter.operator?.value == 'in' || filter.operator?.value == 'not_in') {
		return 'Select one or more values...'
	}
	return 'Type a value...'
})

const applyDisabled = computed(() => isEmptyObj(filter.column, filter.operator, filter.value))

watch(
	() => filter.column,
	(newColumn) => {
		filter.column = newColumn
		filter.operator = {}
		filter.value = {}
	}
)
watch(
	() => filter.operator,
	(newOperator) => {
		filter.operator = newOperator
		filter.value = {}
	}
)

function processListPickerOption(options) {
	if (!Array.isArray(options)) {
		return {
			value: [],
			label: '',
		}
	}
	return {
		value: options.map((option) => (option.hasOwnProperty('value') ? option.value : option)),
		label:
			options.length > 1
				? options.length + ' values'
				: options.length > 0
				? options[0].value || options[0]
				: '',
	}
}

function apply() {
	if (applyDisabled.value) {
		return
	}
	if (showListPicker.value) {
		filter.value = processListPickerOption(filter.value.value)
	} else if (showDatePicker.value) {
		filter.value = {
			value: filter.value.value,
			label: formatDate(filter.value.value),
		}
	}
	emit('filter-select', query.filters.convertIntoExpression(filter))
}

const checkAndFetchColumnValues = debounce(function (search_text = '') {
	if (
		isEmptyObj(filter.column) ||
		!['=', '!=', 'in', 'not_in'].includes(filter.operator?.value)
	) {
		return
	}

	if (filter.column?.type == 'String') {
		query.fetchColumnValues.submit({
			column: filter.column,
			search_text,
		})
	}
}, 300)

watch(
	() => showListPicker.value || showValueOptions.value,
	(show) => {
		// if operator is changed, fetch values for the column
		show && !valueOptions.value?.length && checkAndFetchColumnValues()
	},
	{ immediate: true }
)
</script>
