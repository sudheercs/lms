<template>
	<Dialog
		v-model="show"
		:options="{
			size: '2xl',
			title: __('Bulk Upload Questions'),
		}"
	>
		<template #body-content>
			<div class="space-y-6 py-2">
				<!-- Instructions -->
				<div class="rounded-lg bg-surface-gray-2 px-4 py-3 text-sm text-ink-gray-7 space-y-1">
					<p class="font-semibold text-ink-gray-9">{{ __('How to use') }}</p>
					<ol class="list-decimal list-inside space-y-1">
						<li>{{ __('Download the sample CSV file below.') }}</li>
						<li>{{ __('Fill in your questions following the same column order.') }}</li>
						<li>{{ __('Upload the completed CSV and click Import.') }}</li>
					</ol>
				</div>

				<!-- Download Sample -->
				<div class="flex items-center gap-3">
					<Button variant="outline" @click="downloadSample">
						<template #prefix>
							<Download class="size-4 stroke-1.5" />
						</template>
						{{ __('Download Sample CSV') }}
					</Button>
					<span class="text-xs text-ink-gray-5">
						{{ __('questions: question, type, marks, options 1-4, correct flags, explanations') }}
					</span>
				</div>

				<!-- File Picker -->
				<div>
					<label class="block text-xs font-medium text-ink-gray-5 mb-1.5">
						{{ __('CSV File') }}
					</label>
					<div
						class="flex items-center justify-center gap-3 rounded-lg border-2 border-dashed border-outline-gray-modals bg-surface-gray-1 px-4 py-6 cursor-pointer hover:border-blue-400 transition-colors"
						@click="triggerFileInput"
						@dragover.prevent
						@drop.prevent="onDrop"
					>
						<FileText class="size-8 text-ink-gray-4 shrink-0" />
						<div class="text-center">
							<p v-if="!selectedFile" class="text-sm text-ink-gray-6">
								{{ __('Click to choose a CSV file or drag and drop here') }}
							</p>
							<p v-else class="text-sm font-medium text-ink-gray-9">
								{{ selectedFile.name }}
								<span class="ml-1 text-xs text-ink-gray-5">({{ fileSizeLabel }})</span>
							</p>
						</div>
					</div>
					<input
						ref="fileInputRef"
						type="file"
						accept=".csv,text/csv"
						class="hidden"
						@change="onFileChange"
					/>
				</div>

				<!-- Result Summary -->
				<div v-if="result" class="space-y-3">
					<div
						class="flex items-center gap-2 rounded-lg px-4 py-3 text-sm font-medium"
						:class="
							result.inserted > 0
								? 'bg-green-50 text-green-700'
								: 'bg-yellow-50 text-yellow-700'
						"
					>
						<CheckCircle v-if="result.inserted > 0" class="size-4 shrink-0" />
						<AlertCircle v-else class="size-4 shrink-0" />
						{{ result.inserted }} {{ __('question(s) imported successfully.') }}
					</div>

					<div v-if="result.errors && result.errors.length" class="space-y-1">
						<p class="text-xs font-semibold text-ink-gray-7">
							{{ result.errors.length }} {{ __('row(s) had errors:') }}
						</p>
						<div
							v-for="err in result.errors"
							:key="err.row"
							class="flex gap-2 rounded bg-red-50 px-3 py-1.5 text-xs text-red-700"
						>
							<span class="font-semibold shrink-0">{{ __('Row') }} {{ err.row }}:</span>
							<span>{{ err.message }}</span>
						</div>
					</div>
				</div>
			</div>
		</template>

		<template #actions>
			<div class="flex items-center justify-end gap-2 w-full">
				<Button variant="subtle" @click="show = false">{{ __('Cancel') }}</Button>
				<Button
					variant="solid"
					:disabled="!selectedFile || isUploading"
					:loading="isUploading"
					@click="uploadCSV"
				>
					<template #prefix>
						<Upload class="size-4 stroke-1.5" />
					</template>
					{{ __('Import') }}
				</Button>
			</div>
		</template>
	</Dialog>
</template>

<script setup>
import { Dialog, Button, createResource, toast } from 'frappe-ui'
import { ref, computed } from 'vue'
import {
	Download,
	Upload,
	FileText,
	CheckCircle,
	AlertCircle,
} from 'lucide-vue-next'

const show = defineModel()
const emit = defineEmits(['success'])

const props = defineProps({
	quizID: {
		type: String,
		required: true,
	},
})

// ── State ────────────────────────────────────────────────────────────────────
const fileInputRef = ref(null)
const selectedFile = ref(null)
const isUploading = ref(false)
const result = ref(null)

// ── Computed ─────────────────────────────────────────────────────────────────
const fileSizeLabel = computed(() => {
	if (!selectedFile.value) return ''
	const bytes = selectedFile.value.size
	if (bytes < 1024) return `${bytes} B`
	if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
	return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
})

// ── File picking ──────────────────────────────────────────────────────────────
const triggerFileInput = () => fileInputRef.value?.click()

const onFileChange = (e) => {
	const file = e.target.files?.[0]
	if (file) setFile(file)
}

const onDrop = (e) => {
	const file = e.dataTransfer.files?.[0]
	if (file && (file.name.endsWith('.csv') || file.type === 'text/csv')) {
		setFile(file)
	} else {
		toast.error(__('Please drop a .csv file.'))
	}
}

const setFile = (file) => {
	selectedFile.value = file
	result.value = null
}

// ── Sample CSV ────────────────────────────────────────────────────────────────
const SAMPLE_ROWS = [
	[
		'question',
		'type',
		'marks',
		'option_1',
		'is_correct_1',
		'explanation_1',
		'option_2',
		'is_correct_2',
		'explanation_2',
		'option_3',
		'is_correct_3',
		'explanation_3',
		'option_4',
		'is_correct_4',
		'explanation_4',
	],
	[
		'What is 2 + 2?',
		'Choices',
		'1',
		'3',
		'0',
		'Not correct',
		'4',
		'1',
		'Correct!',
		'5',
		'0',
		'',
		'6',
		'0',
		'',
	],
	[
		'Capital of France?',
		'Choices',
		'2',
		'Berlin',
		'0',
		'',
		'Paris',
		'1',
		'Correct!',
		'Rome',
		'0',
		'',
		'Madrid',
		'0',
		'',
	],
	[
		'Which planet is closest to the Sun?',
		'Choices',
		'1',
		'Venus',
		'0',
		'',
		'Earth',
		'0',
		'',
		'Mercury',
		'1',
		'Mercury is closest',
		'Mars',
		'0',
		'',
	],
]

const downloadSample = () => {
	const csvContent = SAMPLE_ROWS.map((row) =>
		row.map((cell) => (cell.includes(',') ? `"${cell}"` : cell)).join(',')
	).join('\n')

	const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
	const url = URL.createObjectURL(blob)
	const link = document.createElement('a')
	link.href = url
	link.download = 'quiz_questions_sample.csv'
	link.click()
	URL.revokeObjectURL(url)
}

// ── Upload ────────────────────────────────────────────────────────────────────
const uploadResource = createResource({
	url: 'lms.lms.api.bulk_upload_quiz_questions',
	makeParams(values) {
		return {
			quiz: props.quizID,
			filedata: values.filedata,
		}
	},
})

const uploadCSV = () => {
	if (!selectedFile.value) return

	isUploading.value = true
	result.value = null

	const reader = new FileReader()
	reader.onload = (e) => {
		const filedata = e.target.result

		uploadResource.submit(
			{ filedata },
			{
				onSuccess(data) {
					isUploading.value = false
					result.value = data
					if (data.inserted > 0) {
						toast.success(
							__(`${data.inserted} question(s) imported successfully.`)
						)
						emit('success')
					}
					if (!data.errors || data.errors.length === 0) {
						// All clean — close automatically only if no errors
						if (data.inserted > 0) show.value = false
					}
				},
				onError(err) {
					isUploading.value = false
					toast.error(err.messages?.[0] || err)
				},
			}
		)
	}
	reader.onerror = () => {
		isUploading.value = false
		toast.error(__('Failed to read the file.'))
	}
	reader.readAsText(selectedFile.value)
}
</script>
