<template>
	<div class="flex flex-col">
		<!-- Expression Code Field -->
		<div class="flex justify-between">
			<div class="mb-1 text-sm font-light">Expression</div>
			<Tooltip v-if="expression.error" :text="expression.error">
				<div class="!mt-1 flex cursor-pointer items-center text-xs text-red-500">
					<FeatherIcon name="alert-circle" class="h-4 w-4" />
				</div>
			</Tooltip>
		</div>
		<Popover class="h-36 w-full text-sm" placement="left-start">
			<template #target="{ open }">
				<Code
					v-model="input.value"
					:completions="getCompletions"
					@inputChange="open"
					@viewUpdate="codeViewUpdate"
				></Code>
			</template>
			<template #body>
				<div class="w-full pr-3 text-base">
					<transition
						enter-active-class="transition duration-100 ease-out"
						enter-from-class="transform scale-95 opacity-0"
						enter-to-class="transform scale-100 opacity-100"
						leave-active-class="transition duration-75 ease-in"
						leave-from-class="transform opacity-100"
						leave-to-class="transform opacity-0"
					>
						<div
							v-show="expression.help"
							class="ml-auto w-[20rem] rounded-md border bg-white p-2 shadow-lg"
						>
							<span class="mr-1 font-light">Syntax:</span>
							<span class="font-medium italic" style="font-family: 'Fira Code'">
								{{ expression.help?.syntax }}
							</span>
							<br />
							<br />
							<span>{{ expression.help?.description }}</span>
							<br />
							<br />
							<span class="mr-1 font-light">Example:</span>
							<span class="font-medium" style="font-family: 'Fira Code'">
								{{ expression.help?.example }}
							</span>
						</div>
					</transition>
				</div>
			</template>
		</Popover>

		<!-- Label Field -->
		<div class="mt-2 text-sm text-gray-600">
			<div class="mb-1 font-light">Label</div>
			<Input
				type="text"
				v-model="expression.label"
				class="h-8 placeholder:text-sm"
				placeholder="Enter a label..."
			/>
		</div>
		<!-- Type Field -->
		<div class="mt-2 text-sm text-gray-600">
			<div class="mb-1 font-light">Type</div>
			<Input
				type="select"
				v-model="expression.valueType"
				class="h-8 placeholder:text-sm"
				placeholder="Select a type..."
				:options="columnTypes"
			/>
		</div>
		<div v-if="showDateFormatOptions" class="mt-2 text-sm text-gray-600">
			<div class="mb-1 font-light">Date Format</div>
			<Autocomplete
				v-model="expression.dateFormat"
				:options="dateFormats"
				placeholder="Select a date format..."
			/>
		</div>
		<div class="mt-2 space-y-1 text-sm text-gray-600">
			<div class="font-light">Sort</div>
			<Input
				type="select"
				v-model="expression.order_by"
				:options="[
					{
						label: '',
						value: '',
					},
					{
						label: 'Ascending',
						value: 'asc',
					},
					{
						label: 'Descending',
						value: 'desc',
					},
				]"
				class="h-8 placeholder:text-sm"
				placeholder="Enter a label..."
			/>
		</div>
		<div class="mt-4 text-sm text-gray-600">
			<Input
				v-if="expression.valueType == 'String'"
				type="checkbox"
				label="Group By"
				v-model="expression.groupBy"
			/>
		</div>
		<!-- Action Buttons -->
		<div class="mt-3 flex justify-end space-x-2">
			<Button
				v-if="editing"
				class="text-red-500"
				appearance="white"
				@click="removeExpressionColumn"
			>
				Remove
			</Button>
			<Button appearance="primary" @click="addOrEditColumn" :disabled="addDisabled">
				{{ editing ? 'Update' : 'Add ' }}
			</Button>
		</div>
	</div>
</template>

<script setup>
import Autocomplete from '@/components/Controls/Autocomplete.vue'
import Code from '@/components/Controls/Code.vue'
import Tooltip from '@/components/Tooltip.vue'
import { debounce } from 'frappe-ui'

import { dateFormats } from '@/utils/format'
import { FUNCTIONS } from '@/utils/query'
import { parse } from '@/utils/expressions'
import { ref, inject, watchEffect, reactive, computed } from 'vue'

const query = inject('query')

const emit = defineEmits(['close'])
const props = defineProps({
	column: {
		type: Object,
		default: {},
		validate: (value) => {
			if (value.is_expression != 1) {
				return 'Column must be an expression'
			}
		},
	},
})

const column = {
	...props.column,
	expression: props.column.expression || {},
}
const editing = ref(Boolean(column.name))
const input = reactive({
	value: column.expression.raw || '',
	caretPosition: column.expression.raw?.length || 0,
})

const columnTypes = ['String', 'Integer', 'Decimal', 'Text', 'Datetime', 'Date', 'Time']

// parse the expression when input changes
const expression = reactive({
	raw: input.value,
	label: column.label,
	order_by: column.order_by,
	groupBy: column.aggregation == 'Group By',
	valueType: column.type || 'String',
	ast: null,
	error: null,
	tokens: [],
	help: null,
	dateFormat: ['Date', 'Datetime'].includes(props.column.type)
		? dateFormats.find((format) => format.value === props.column.format_option?.date_format)
		: {},
})
watchEffect(() => {
	expression.raw = input.value
	const { ast, tokens, errorMessage } = parse(input.value)
	expression.ast = ast
	expression.tokens = tokens
	expression.error = errorMessage
})
const showDateFormatOptions = computed(() => ['Date', 'Datetime'].includes(expression.valueType))
watchEffect(() => {
	if (showDateFormatOptions.value || expression.valueType !== 'String') {
		// Currently group by date field is not supported on expressions due to.
		// pymysql.err.OperationalError: (1056, "Can't group on '{AGGREGATE} of {DATE_FIELD}'")
		expression.groupBy = false
	}
})

const codeViewUpdate = debounce(function ({ cursorPos }) {
	expression.help = null

	const { tokens } = expression
	const token = tokens
		.filter((t) => t.start <= cursorPos - 1 && t.end >= cursorPos && t.type == 'FUNCTION')
		.at(-1)
	if (token) {
		const { value } = token
		if (FUNCTIONS[value]) {
			expression.help = FUNCTIONS[value]
		}
	}
}, 300)

const getCompletions = (context, syntaxTree) => {
	let word = context.matchBefore(/\w*/)
	let nodeBefore = syntaxTree.resolveInner(context.pos, -1)

	if (nodeBefore.name === 'TemplateString') {
		return {
			from: word.from,
			options: query.columns.options.map((c) => {
				return { label: `${c.table}.${c.column}` }
			}),
		}
	}
	if (nodeBefore.name === 'VariableName') {
		return {
			from: word.from,
			options: Object.keys(FUNCTIONS).map((label) => ({ label })),
		}
	}
}

const addDisabled = computed(() => {
	return Boolean(
		expression.error || !expression.raw || !expression.label || !expression.valueType
	)
})

const addOrEditColumn = () => {
	const newColumn = {
		name: props.column.name,
		is_expression: 1,
		expression: {
			raw: expression.raw,
			ast: expression.ast,
		},
		type: expression.valueType,
		label: expression.label,
		order_by: expression.order_by,
		column: expression.label.replace(/\s/g, '_'),
		aggregation: expression.groupBy ? 'Group By' : '',
	}

	if (showDateFormatOptions.value) {
		newColumn.format_option = {
			date_format: expression.dateFormat.value,
		}
	}

	if (props.column.name) {
		query.updateColumn.submit({ column: newColumn })
	} else {
		query.addColumn.submit({ column: newColumn })
	}
	emit('close')
}

const removeExpressionColumn = () => {
	query.removeColumn.submit({ column: props.column })
	emit('close')
}
</script>
