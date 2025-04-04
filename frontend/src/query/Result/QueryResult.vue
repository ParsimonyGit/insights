<template>
	<div
		ref="resultContainer"
		class="flex w-full flex-1 flex-shrink-0 flex-col overflow-hidden bg-white p-2"
	>
		<!-- Result Header -->
		<div class="relative flex h-8 flex-shrink-0 items-center justify-between">
			<div class="flex space-x-1">
				<div class="text-sm tracking-wide text-gray-600">RESULT</div>
				<div
					v-if="executionTime"
					class="flex items-center space-x-1 text-sm font-light text-gray-500"
				>
					<span class="text-sm text-gray-500">
						({{ totalRows.toLocaleString() }} rows in {{ executionTime }}s)
					</span>
					<Tooltip
						v-if="totalRows > query.results.MAX_ROWS"
						:text="`Results are limited to ${query_result_limit} rows. You can change this in the settings.`"
						:hoverDelay="0.1"
						class="flex"
					>
						<FeatherIcon name="info" class="h-3 w-3 cursor-pointer" />
					</Tooltip>
				</div>
			</div>
			<LimitsAndOrder class="-mt-1" />
		</div>
		<!-- Result  -->
		<div class="relative flex flex-1 overflow-hidden">
			<!-- Empty State -->
			<div
				v-if="!needsExecution && formattedResult?.length === 0"
				class="flex h-full w-full items-center justify-center rounded-md border-2 border-dashed border-gray-200 font-light text-gray-400"
			>
				<p>No results found</p>
			</div>
			<div v-else class="relative flex flex-1 select-text flex-col-reverse overflow-hidden">
				<div class="flex-1 overflow-scroll rounded-md bg-gray-50 pt-0 scrollbar-hide">
					<table class="border-separate border-spacing-0 text-sm">
						<thead class="sticky top-0 text-gray-600">
							<tr>
								<th
									v-for="column in columns"
									:key="column.name"
									class="whitespace-nowrap border-b border-r bg-gray-100 px-3 py-1.5 font-medium text-gray-700"
									scope="col"
								>
									<ColumnHeader :column="column" :query="query" />
								</th>
								<th
									class="border-b bg-gray-100 px-3 py-1.5 font-medium text-gray-700"
									scope="col"
									width="99%"
								></th>
							</tr>
						</thead>
						<tbody
							class="pointer-events-none"
							:class="{ 'blur-[2px]': needsExecution }"
						>
							<tr v-for="(row, i) in formattedResult" :key="i">
								<td
									v-for="(cell, j) in row"
									:key="j"
									class="whitespace-nowrap border-b border-r bg-gray-50 px-3 py-2 text-gray-600"
									:class="{ 'text-right': isNumberColumn[j] }"
								>
									{{
										typeof cell == 'number'
											? cell.toLocaleString()
											: ellipsis(cell, 100)
									}}
								</td>
								<td
									class="border-b bg-gray-50 px-3 py-2 text-gray-600"
									width="99%"
								></td>
							</tr>
						</tbody>
					</table>

					<!-- If Pending Execution -->
					<div
						v-if="query.run.loading || needsExecution"
						class="absolute top-0 left-0 flex h-full w-full items-center justify-center"
					>
						<Button
							appearance="primary"
							class="!shadow-md"
							@click="query.debouncedRun()"
							:loading="query.run.loading"
							loadingText="Executing..."
						>
							{{ query.run.loading ? '' : 'Execute' }}
						</Button>
					</div>
				</div>
			</div>
		</div>
	</div>
</template>

<script setup>
import LimitsAndOrder from '@/query/LimitsAndOrder.vue'
import ColumnHeader from '@/query/Result/ColumnHeader.vue'
import { ellipsis, FIELDTYPES } from '@/utils'
import settings from '@/utils/settings'

import { computed, inject, watch } from 'vue'

const query = inject('query')

const query_result_limit = computed(() =>
	parseInt(settings.doc?.query_result_limit).toLocaleString()
)
const formattedResult = computed(() => query.results.formattedResult.slice(1))
const needsExecution = computed(() => query.doc?.status === 'Pending Execution')
const columns = computed(() => {
	return query.results.formattedResult[0]?.map((c) => c.split('::')[0])
})
const isNumberColumn = computed(() => {
	return query.doc.columns.map((c) => FIELDTYPES.NUMBER.includes(c.type))
})

if (settings.doc?.auto_execute_query) {
	watch(needsExecution, (newVal, oldVal) => newVal && !oldVal && query.execute(), {
		immediate: true,
	})
}

const executionTime = computed(() => {
	const rounded = Math.round(query.doc.execution_time * 100) / 100
	return query.doc.execution_time && rounded < 0.01 ? '< 0.01' : rounded
})

const totalRows = computed(() => {
	return query.doc.results_row_count - 1
})
</script>
