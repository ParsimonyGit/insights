<template>
	<div class="relative flex flex-1 overflow-hidden rounded-md border px-2 py-3">
		<div
			v-if="props.rows.length == 0"
			class="absolute top-0 flex h-full w-full items-center justify-center text-lg font-light text-gray-500"
		>
			<span>No Data</span>
		</div>
		<div class="flex flex-1 flex-col overflow-scroll text-base scrollbar-hide">
			<table>
				<thead class="sticky top-0" v-if="props.header">
					<tr>
						<slot name="header">
							<td
								v-for="head in props.header"
								class="whitespace-nowrap bg-gray-100 px-2.5 py-1.5 font-medium text-gray-600 first:rounded-l-md last:rounded-r-md"
								scope="col"
							>
								{{ head }}
							</td>
						</slot>
					</tr>
				</thead>
				<tbody>
					<tr v-for="(row, index) in props.rows" class="border-b">
						<td
							v-for="cell in row"
							class="whitespace-nowrap bg-white px-2.5 py-2 font-light text-gray-600"
						>
							{{ ellipsis(cell, 100) }}
						</td>
					</tr>
				</tbody>
			</table>
		</div>
	</div>
</template>

<script setup>
import { ellipsis } from '@/utils'

const props = defineProps(['header', 'rows'])
</script>
