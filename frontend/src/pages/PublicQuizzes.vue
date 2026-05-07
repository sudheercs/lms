<template>
	<header class="sticky top-0 z-10 flex items-center justify-between border-b bg-white px-5 py-2.5">
		<Breadcrumbs :items="breadcrumbs" />
	</header>

	<div class="px-5 pt-6 pb-10">
		<div class="mb-6">
			<h1 class="text-2xl font-bold text-gray-900">{{ __('Available Tests') }}</h1>
			<p class="text-sm text-gray-500 mt-1">{{ __('Select a test below to begin. No enrollment required.') }}</p>
		</div>

		<div class="mb-6 max-w-sm">
			<FormControl v-model="search" type="text" :placeholder="__('Search tests…')">
				<template #prefix>
					<Search class="size-4 text-gray-400" />
				</template>
			</FormControl>
		</div>

		<div v-if="quizzes.loading" class="flex justify-center py-20">
			<div class="animate-spin w-8 h-8 border-4 border-blue-600 border-t-transparent rounded-full"></div>
		</div>

		<div v-else-if="!filteredQuizzes.length" class="text-center py-20 text-gray-400">
			<ClipboardList class="w-12 h-12 mx-auto mb-3 opacity-40" />
			<p class="text-lg font-medium">{{ __('No tests available') }}</p>
			<p class="text-sm mt-1">{{ __('Check back later or contact your administrator.') }}</p>
		</div>

		<div v-else class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-5">
			<div
				v-for="quiz in filteredQuizzes"
				:key="quiz.name"
				class="bg-white border border-gray-200 rounded-xl shadow-sm hover:shadow-md transition-shadow flex flex-col"
			>
				<div class="bg-gradient-to-r from-blue-600 to-indigo-600 rounded-t-xl px-4 py-4">
					<h2 class="text-base font-bold text-white leading-tight line-clamp-2">{{ quiz.title }}</h2>
					<div v-if="quiz.modules?.length" class="flex flex-wrap gap-1 mt-2">
						<span
							v-for="mod in quiz.modules"
							:key="mod"
							class="bg-white/20 text-white text-xs px-2 py-0.5 rounded-full"
						>{{ mod }}</span>
					</div>
				</div>

				<div class="flex-1 px-4 py-4 space-y-2">
					<div class="grid grid-cols-2 gap-2 text-sm">
						<div class="flex items-center gap-1.5 text-gray-600">
							<Hash class="w-4 h-4 text-gray-400" />
							<span>{{ quiz.question_count }} {{ __('Questions') }}</span>
						</div>
						<div class="flex items-center gap-1.5 text-gray-600">
							<Award class="w-4 h-4 text-yellow-500" />
							<span>{{ quiz.total_marks }} {{ __('Marks') }}</span>
						</div>
						<div class="flex items-center gap-1.5 text-gray-600">
							<BarChart2 class="w-4 h-4 text-green-500" />
							<span>{{ __('Pass: {0}%').format(quiz.passing_percentage) }}</span>
						</div>
						<div v-if="quiz.duration" class="flex items-center gap-1.5 text-gray-600">
							<Clock class="w-4 h-4 text-blue-500" />
							<span>{{ quiz.duration }} {{ __('min') }}</span>
						</div>
						<div v-if="quiz.max_attempts" class="flex items-center gap-1.5 text-gray-600">
							<RefreshCw class="w-4 h-4 text-purple-500" />
							<span>{{ quiz.max_attempts }} {{ __('attempt(s)') }}</span>
						</div>
					</div>
				</div>

				<div class="px-4 pb-4">
					<router-link :to="{ name: 'QuizPage', params: { quizID: quiz.name } }">
						<button class="w-full py-2 bg-blue-600 hover:bg-blue-700 text-white text-sm font-semibold rounded-lg transition-colors">
							{{ __('Start Test') }}
						</button>
					</router-link>
				</div>
			</div>
		</div>
	</div>
</template>

<script setup>
import { Breadcrumbs, FormControl, createResource, usePageMeta } from 'frappe-ui'
import { computed, ref } from 'vue'
import { Search, ClipboardList, Hash, Award, BarChart2, Clock, RefreshCw } from 'lucide-vue-next'
import { sessionStore } from '@/stores/session'

const { brand } = sessionStore()
const search = ref('')

const quizzes = createResource({
	url: 'lms.lms.api.get_published_quizzes',
	auto: true,
})

const filteredQuizzes = computed(() => {
	const q = search.value.toLowerCase()
	return (quizzes.data || []).filter((quiz) =>
		quiz.title.toLowerCase().includes(q) ||
		(quiz.modules || []).some((m) => m.toLowerCase().includes(q))
	)
})

const breadcrumbs = computed(() => [{ label: __('Tests'), route: { name: 'PublicQuizzes' } }])

usePageMeta(() => ({ title: __('Available Tests'), icon: brand.favicon }))
</script>
