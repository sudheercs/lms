<template>
	<!-- Pre-quiz start screen -->
	<div v-if="quiz.data && activeQuestion === 0" class="max-w-2xl mx-auto my-10 border rounded-lg p-8 text-center space-y-4">
		<div class="text-xl font-bold text-gray-900">{{ quiz.data.title }}</div>
		<ul class="text-sm text-left text-gray-700 bg-blue-50 rounded-lg p-4 space-y-1 list-decimal list-inside">
			<li>{{ __('Do not refresh the page or close this window.') }}</li>
			<li>{{ __('This quiz consists of {0} questions.').format(questions.length) }}</li>
			<li v-if="quiz.data?.duration">{{ __('Complete all questions in {0} minutes.').format(quiz.data.duration) }}</li>
			<li v-if="quiz.data?.passing_percentage">{{ __('You need {0}% to pass.').format(quiz.data.passing_percentage) }}</li>
			<li v-if="quiz.data?.max_attempts">{{ __('Maximum attempts: {0}').format(quiz.data.max_attempts) }}</li>
			<li v-if="quiz.data?.enable_negative_marking">{{ __('Negative marking: {0} mark(s) deducted per wrong answer.').format(quiz.data.marks_to_cut) }}</li>
		</ul>
		<div v-if="quiz.data.max_attempts && attempts.data?.length >= quiz.data.max_attempts" class="text-red-600 text-sm">
			{{ __('You have exceeded the maximum number of attempts for this quiz.') }}
		</div>
		<div v-else class="flex justify-center gap-3">
			<Button variant="solid" @click="startQuiz">{{ inVideo ? __('Start the Quiz') : __('Start') }}</Button>
			<Button v-if="inVideo" @click="props.backToVideo()">{{ __('Resume Video') }}</Button>
		</div>
	</div>

	<!-- Active NTA-style quiz -->
	<div v-else-if="quiz.data && !quizSubmission.data && activeQuestion > 0" class="flex flex-col" style="height:calc(100vh - 53px)">
		<!-- Header -->
		<div class="flex items-center justify-between bg-white border-b px-4 py-2 shadow-sm flex-shrink-0">
			<div class="flex items-center gap-3">
				<div class="w-14 h-14 bg-gray-200 border-2 border-gray-400 rounded flex items-center justify-center">
					<svg class="w-8 h-8 text-gray-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z"/></svg>
				</div>
				<div class="text-sm leading-6">
					<div>Candidate Name : <span class="text-orange-500 font-semibold">{{ user.data?.full_name }}</span></div>
					<div>Exam Name : <span class="text-green-700 font-semibold">{{ quiz.data?.title }}</span></div>
					<div v-if="quiz.data?.duration">Remaining Time : <span class="bg-blue-600 text-white px-2 py-0.5 rounded font-mono text-xs ml-1">{{ formatTimer(timer) }}</span></div>
				</div>
			</div>
			<select class="border border-gray-300 rounded px-3 py-1.5 text-sm bg-white"><option>English</option></select>
		</div>

		<!-- Body -->
		<div class="flex flex-1 overflow-hidden">
			<!-- Left: Question panel -->
			<div class="flex-1 flex flex-col overflow-hidden bg-white">
				<div class="flex-1 overflow-y-auto p-6">
					<div v-for="(question, idx) in questions" :key="idx">
						<div v-if="idx === activeQuestion - 1 && questionDetails.data">
							<div class="flex items-center justify-between mb-4">
								<h2 class="text-base font-bold text-gray-800">Question {{ activeQuestion }} :</h2>
								<span class="bg-blue-100 text-blue-700 px-2 py-1 rounded text-xs font-medium">{{ question.marks }} {{ question.marks == 1 ? 'Mark' : 'Marks' }}</span>
							</div>
							<div class="text-sm text-gray-800 leading-relaxed mb-6" v-html="questionDetails.data.question"></div>
							<!-- Choices -->
							<div v-if="questionDetails.data.type === 'Choices'" class="space-y-3">
								<template v-for="n in 4" :key="n">
									<label v-if="questionDetails.data[`option_${n}`]" class="flex items-center gap-3 border rounded p-3 cursor-pointer transition-all" :class="selectedOptions[n-1] ? 'border-blue-500 bg-blue-50' : 'border-gray-300 hover:bg-gray-50'">
										<input v-if="!showAnswers.length && !questionDetails.data.multiple" type="radio" :name="'q'+activeQuestion" :checked="selectedOptions[n-1]" @change="markAnswer(n)" class="w-4 h-4 accent-blue-600" />
										<input v-else-if="!showAnswers.length && questionDetails.data.multiple" type="checkbox" :checked="selectedOptions[n-1]" @change="markAnswer(n)" class="w-4 h-4 accent-blue-600 rounded" />
										<template v-else-if="quiz.data.show_answers">
											<CheckCircle v-if="showAnswers[n-1] == 1" class="w-4 h-4 text-green-600 shrink-0" />
											<MinusCircle v-else-if="showAnswers[n-1] == 2" class="w-4 h-4 text-green-600 shrink-0" />
											<XCircle v-else-if="showAnswers[n-1] == 0" class="w-4 h-4 text-red-500 shrink-0" />
											<MinusCircle v-else class="w-4 h-4 shrink-0" />
										</template>
										<span class="text-sm text-gray-800">{{ n }}.&nbsp;&nbsp;{{ questionDetails.data[`option_${n}`] }}</span>
									</label>
								</template>
							</div>
							<!-- User Input -->
							<div v-else-if="questionDetails.data.type === 'User Input'">
								<FormControl v-model="possibleAnswer" type="textarea" :disabled="!!showAnswers.length" class="mt-2" />
								<div v-if="showAnswers.length" class="mt-2"><Badge v-if="showAnswers[0]" :label="__('Correct')" theme="green" /><Badge v-else theme="red" :label="__('Incorrect')" /></div>
							</div>
							<!-- Open Ended -->
							<div v-else>
								<TextEditor class="mt-4" :content="possibleAnswer" @change="(v) => (possibleAnswer = v)" :editable="true" :fixedMenu="true" editorClass="prose-sm max-w-none border-b border-x border-gray-300 bg-gray-50 rounded-b-md py-1 px-2 min-h-[7rem]" />
							</div>
						</div>
					</div>
				</div>
				<!-- Action buttons -->
				<div class="border-t bg-gray-50 px-6 py-3 flex-shrink-0 space-y-3">
					<div class="flex flex-wrap gap-2">
						<button @click="saveAndNext()" class="px-4 py-2 bg-green-600 hover:bg-green-700 text-white text-xs font-bold rounded uppercase">Save &amp; Next</button>
						<button @click="clearAnswer()" class="px-4 py-2 bg-white hover:bg-gray-100 text-gray-700 text-xs font-bold border border-gray-400 rounded uppercase">Clear</button>
						<button @click="saveAndMarkForReview()" class="px-4 py-2 bg-orange-500 hover:bg-orange-600 text-white text-xs font-bold rounded uppercase">Save &amp; Mark for Review</button>
						<button @click="markForReviewAndNext()" class="px-4 py-2 bg-purple-600 hover:bg-purple-700 text-white text-xs font-bold rounded uppercase">Mark for Review &amp; Next</button>
					</div>
					<div class="flex items-center justify-between">
						<div class="flex gap-2">
							<button @click="switchQuestion(activeQuestion - 1)" :disabled="activeQuestion <= 1" class="px-3 py-1.5 border border-gray-400 text-xs font-semibold rounded hover:bg-gray-100 disabled:opacity-40">&lt;&lt; Back</button>
							<button @click="switchQuestion(activeQuestion + 1)" :disabled="activeQuestion >= questions.length" class="px-3 py-1.5 border border-gray-400 text-xs font-semibold rounded hover:bg-gray-100 disabled:opacity-40">Next &gt;&gt;</button>
						</div>
						<button @click="handleSubmitClick()" class="px-5 py-2 bg-green-600 hover:bg-green-700 text-white text-xs font-bold rounded uppercase">Submit</button>
					</div>
				</div>
			</div>

			<!-- Right: Question palette -->
			<div class="w-72 bg-gray-50 border-l flex flex-col overflow-hidden flex-shrink-0">
				<!-- Legend -->
				<div class="p-3 border-b bg-white space-y-2 flex-shrink-0">
					<div class="grid grid-cols-2 gap-2">
						<div class="flex items-center gap-1.5 text-xs"><span class="w-7 h-7 rounded-full bg-gray-400 text-white flex items-center justify-center font-bold text-xs shrink-0">{{ notVisitedCount }}</span><span class="text-gray-600">Not Visited</span></div>
						<div class="flex items-center gap-1.5 text-xs"><span class="w-7 h-7 rounded-full bg-red-500 text-white flex items-center justify-center font-bold text-xs shrink-0">{{ notAnsweredCount }}</span><span class="text-gray-600">Not Answered</span></div>
						<div class="flex items-center gap-1.5 text-xs"><span class="w-7 h-7 rounded-full bg-green-600 text-white flex items-center justify-center font-bold text-xs shrink-0">{{ answeredCount }}</span><span class="text-gray-600">Answered</span></div>
						<div class="flex items-center gap-1.5 text-xs"><span class="w-7 h-7 rounded-full bg-purple-600 text-white flex items-center justify-center font-bold text-xs shrink-0">{{ markedOnlyCount }}</span><span class="text-gray-600">Marked for Review</span></div>
					</div>
					<div class="flex items-center gap-1.5 text-xs"><span class="relative w-7 h-7 rounded-full bg-purple-600 text-white flex items-center justify-center font-bold text-xs shrink-0">{{ answeredAndMarkedCount }}<span class="absolute -bottom-0.5 -right-0.5 w-3 h-3 bg-green-500 rounded-full border-2 border-white"></span></span><span class="text-gray-600 leading-tight">Answered &amp; Marked for Review</span></div>
				</div>
				<!-- Grid -->
				<div class="flex-1 overflow-y-auto p-3">
					<div class="grid grid-cols-7 gap-1.5">
						<button v-for="(q, idx) in questions" :key="idx" @click="switchQuestion(idx + 1)" class="w-8 h-8 rounded text-xs font-bold flex items-center justify-center transition-all" :class="getQuestionBtnClass(idx + 1)">{{ String(idx + 1).padStart(2, '0') }}</button>
					</div>
				</div>
			</div>
		</div>
	</div>

	<!-- Result screen -->
	<div v-else-if="quiz.data && quizSubmission.data" class="border rounded-lg p-20 text-center space-y-4">
		<div class="text-lg font-semibold text-gray-900">{{ __('Quiz Summary') }}</div>
		<div v-if="quizSubmission.data.is_open_ended" class="text-gray-600 leading-5">{{ __("Your submission has been saved. The instructor will review and grade it shortly.") }}</div>
		<div v-else class="text-gray-700">{{ __('You got {0}% correct answers with a score of {1} out of {2}').format(Math.ceil(quizSubmission.data.percentage), quizSubmission.data.score, quizSubmission.data.score_out_of) }}</div>
		<div class="flex gap-2 justify-center">
			<Button @click="resetQuiz()" v-if="!quiz.data.max_attempts || attempts?.data.length < quiz.data.max_attempts">{{ __('Try Again') }}</Button>
			<Button v-if="inVideo" @click="props.backToVideo()">{{ __('Resume Video') }}</Button>
		</div>
		<div v-if="quiz.data.show_submission_history && attempts?.data?.length > 0" class="mt-10">
			<ListView :columns="getSubmissionColumns()" :rows="attempts?.data" row-key="name" :options="{ selectable: false, showTooltip: false }" />
		</div>
	</div>

	<Dialog v-model="showSubmissionConfirmation" :options="{ title: __('Are you sure you want to submit the quiz?'), actions: [{ size: 'sm', label: __('Submit'), variant: 'solid', onClick() { submitQuiz(); showSubmissionConfirmation = false } }] }">
		<template #body-content>
			<div class="border border-gray-200 rounded-lg text-base divide-y">
				<div class="grid grid-cols-2 divide-x"><div class="p-2">{{ __('Total Questions') }}</div><div class="p-2">{{ questions.length }}</div></div>
				<div class="grid grid-cols-2 divide-x"><div class="p-2">{{ __('Attempted Questions') }}</div><div class="p-2">{{ attemptedQuestions.length }}</div></div>
				<div class="grid grid-cols-2 divide-x"><div class="p-2">{{ __('Unattempted Questions') }}</div><div class="p-2">{{ questions.length - attemptedQuestions.length }}</div></div>
			</div>
		</template>
	</Dialog>
</template>
<script setup>
import {
	Badge,
	Button,
	call,
	Checkbox,
	createResource,
	Dialog,
	ListView,
	TextEditor,
	FormControl,
	toast,
} from 'frappe-ui'
import {
	computed,
	inject,
	onMounted,
	onUnmounted,
	reactive,
	ref,
	watch,
} from 'vue'
import {
	CheckCircle,
	ChevronLeft,
	ChevronRight,
	XCircle,
	MinusCircle,
} from 'lucide-vue-next'
import { timeAgo } from '@/utils'
import ProgressBar from '@/components/ProgressBar.vue'

const user = inject('$user')
const activeQuestion = ref(0)
const currentQuestion = ref('')
const selectedOptions = ref([0, 0, 0, 0])
const showAnswers = reactive([])
let questions = reactive([])
const attemptedQuestions = ref([])
const reviewQuestions = ref([])
const showSubmissionConfirmation = ref(false)
const possibleAnswer = ref(null)
const timer = ref(0)
let timerInterval = null

const props = defineProps({
	quizName: {
		type: String,
		required: true,
	},
	inVideo: {
		type: Boolean,
		default: false,
	},
	backToVideo: {
		type: Function,
		default: () => {},
	},
})

onMounted(() => {
	window.addEventListener('pagehide', handlePageHide)
	window.addEventListener('beforeunload', handleBeforeUnload)
})

onUnmounted(() => {
	window.removeEventListener('pagehide', handlePageHide)
	window.removeEventListener('beforeunload', handleBeforeUnload)
})

const handlePageHide = () => {
	if (activeQuestion.value > 0 && !quizSubmission.data) {
		const params = new URLSearchParams({
			quiz: quiz.data.name,
			results: localStorage.getItem(quiz.data.title),
		})

		navigator.sendBeacon(
			'/api/method/lms.lms.doctype.lms_quiz.lms_quiz.submit_quiz?' +
				params.toString()
		)
	}
}

const handleBeforeUnload = (event) => {
	if (activeQuestion.value > 0 && !quizSubmission.data) {
		if (attemptedQuestions.value.length) {
			switchQuestion(activeQuestion.value)
		}
		event.preventDefault()
		event.returnValue = ''
	}
}

const quiz = createResource({
	url: 'frappe.client.get',
	makeParams(values) {
		return {
			doctype: 'LMS Quiz',
			name: props.quizName,
		}
	},
	cache: ['quiz', props.quizName],
	auto: true,
	transform(data) {
		data.duration = parseInt(data.duration)
	},
	onSuccess(data) {
		populateQuestions()
		setupTimer()
	},
})

const populateQuestions = () => {
	let data = quiz.data
	if (data.shuffle_questions) {
		questions = shuffleArray(data.questions)
		if (data.limit_questions_to) {
			questions = questions.slice(0, data.limit_questions_to)
		}
	} else {
		questions = data.questions
	}
}

const setupTimer = () => {
	if (quiz.data.duration) {
		timer.value = quiz.data.duration * 60
	}
}

const startTimer = () => {
	timerInterval = setInterval(() => {
		timer.value--
		if (timer.value == 0) {
			clearInterval(timerInterval)
			submitQuiz()
		}
	}, 1000)
}

const formatTimer = (seconds) => {
	const hrs = Math.floor(seconds / 3600)
		.toString()
		.padStart(2, '0')
	const mins = Math.floor((seconds % 3600) / 60)
		.toString()
		.padStart(2, '0')
	const secs = (seconds % 60).toString().padStart(2, '0')
	return hrs != '00' ? `${hrs}:${mins}:${secs}` : `${mins}:${secs}`
}

const timerProgress = computed(() => {
	return (timer.value / (quiz.data.duration * 60)) * 100
})

const shuffleArray = (array) => {
	for (let i = array.length - 1; i > 0; i--) {
		const j = Math.floor(Math.random() * (i + 1))
		;[array[i], array[j]] = [array[j], array[i]]
	}
	return array
}

const attempts = createResource({
	url: 'frappe.client.get_list',
	makeParams(values) {
		return {
			doctype: 'LMS Quiz Submission',
			filters: {
				member: user.data?.name,
				quiz: quiz.data?.name,
			},
			fields: [
				'name',
				'creation',
				'score',
				'score_out_of',
				'percentage',
				'passing_percentage',
			],
			order_by: 'creation desc',
		}
	},
	transform(data) {
		data.forEach((submission, index) => {
			submission.creation = timeAgo(submission.creation)
			submission.idx = index + 1
		})
	},
})

watch(
	() => quiz.data,
	() => {
		if (quiz.data) {
			populateQuestions()
		}
		if (quiz.data && quiz.data.max_attempts) {
			attempts.reload()
			resetQuiz()
		}
	}
)

const quizSubmission = createResource({
	url: 'lms.lms.doctype.lms_quiz.lms_quiz.submit_quiz',
	makeParams(values) {
		return {
			quiz: quiz.data.name,
			results: localStorage.getItem(quiz.data.title),
		}
	},
})

const questionDetails = createResource({
	url: 'lms.lms.utils.get_question_details',
	makeParams(values) {
		return {
			question: currentQuestion.value,
		}
	},
})

watch(activeQuestion, (value) => {
	if (value > 0) {
		currentQuestion.value = quiz.data.questions[value - 1].question
		questionDetails.reload(
			{},
			{
				onSuccess() {
					if (!quiz.data.show_answers) {
						loadSavedAnswers()
					}
				},
			}
		)
	}
})

const switchQuestion = (questionNumber) => {
	let answers = getAnswers()
	if (answers.length) {
		if (!attemptedQuestions.value.includes(activeQuestion.value)) {
			attemptedQuestions.value.push(activeQuestion.value)
		}
		addToLocalStorage()
		resetQuestion()
	}

	if (questionNumber < 1 || questionNumber > questions.length) return
	activeQuestion.value = questionNumber
}

const loadSavedAnswers = () => {
	let quizData = JSON.parse(localStorage.getItem(quiz.data.title))
	if (quizData) {
		let localQuestion = quizData.find(
			(q) => q.question_name == currentQuestion.value
		)
		if (localQuestion) {
			let localAnswers = localQuestion.answer
			if (localAnswers.length) {
				if (questionDetails.data.type == 'Choices') {
					localAnswers.forEach((answer) => {
						for (let i = 1; i <= 4; i++) {
							if (questionDetails.data[`option_${i}`] == answer) {
								selectedOptions.value[i - 1] = 1
							}
						}
					})
				} else {
					possibleAnswer.value = localAnswers[0]
				}
			}
		}
	}
}

watch(
	() => props.quizName,
	(newName) => {
		if (newName) {
			quiz.reload()
		}
	}
)

const startQuiz = () => {
	activeQuestion.value = 1
	localStorage.removeItem(quiz.data.title)
	if (quiz.data.duration) startTimer()
}

const markAnswer = (index) => {
	if (!questionDetails.data.multiple)
		selectedOptions.value.splice(
			0,
			selectedOptions.value.length,
			...[0, 0, 0, 0]
		)
	selectedOptions.value[index - 1] = selectedOptions.value[index - 1] ? 0 : 1
}

const getAnswers = () => {
	let answers = []
	const type = questionDetails.data.type
	if (type == 'Choices') {
		selectedOptions.value.forEach((value, index) => {
			if (selectedOptions.value[index])
				answers.push(questionDetails.data[`option_${index + 1}`])
		})
	} else {
		answers.push(possibleAnswer.value)
	}

	return answers
}

const checkAnswer = () => {
	let answers = getAnswers()
	if (!answers.length) {
		toast.warning(__('Please select an option'))
		return
	}

	createResource({
		url: 'lms.lms.doctype.lms_quiz.lms_quiz.check_answer',
		params: {
			question: currentQuestion.value,
			question_type: questionDetails.data.type,
			answers: JSON.stringify(answers),
		},
		auto: true,
		onSuccess(data) {
			let type = questionDetails.data.type
			if (type == 'Choices') {
				selectedOptions.value.forEach((option, index) => {
					if (option) {
						showAnswers[index] = option && data[index]
					} else if (data[index] == 2) {
						showAnswers[index] = 2
					} else {
						showAnswers[index] = undefined
					}
				})
			} else {
				showAnswers.push(data)
			}
			addToLocalStorage()
			if (!quiz.data.show_answers) {
				resetQuestion()
			}
		},
	})
}

const addToLocalStorage = () => {
	let quizData = JSON.parse(localStorage.getItem(quiz.data.title))
	let questionData = {
		question_name: currentQuestion.value,
		answer: getAnswers(),
	}
	if (quizData) {
		let existingQuestion = quizData.find(
			(q) => q.question_name == questionData.question_name
		)
		if (existingQuestion) {
			existingQuestion.answer = questionData.answer
		} else {
			quizData.push(questionData)
		}
	} else {
		quizData = [questionData]
	}
	localStorage.setItem(quiz.data.title, JSON.stringify(quizData))
}

const nextQuestion = () => {
	if (!quiz.data.show_answers) return
	if (questionDetails.data?.type == 'Open Ended') addToLocalStorage()
	resetQuestion()
}

const resetQuestion = () => {
	if (activeQuestion.value == quiz.data.questions.length) return
	activeQuestion.value = activeQuestion.value + 1
	selectedOptions.value.splice(0, selectedOptions.value.length, ...[0, 0, 0, 0])
	showAnswers.length = 0
	possibleAnswer.value = null
}

const submitQuiz = () => {
	if (!quiz.data.show_answers) {
		if (questionDetails.data.type == 'Open Ended') addToLocalStorage()
		setTimeout(() => {
			createSubmission()
		}, 500)
		return
	}
	createSubmission()
}

const createSubmission = () => {
	quizSubmission.submit(
		{},
		{
			onSuccess(data) {
				markLessonProgress()
				if (quiz.data && quiz.data.max_attempts) attempts.reload()
				if (quiz.data.duration) clearInterval(timerInterval)
			},
			onError(err) {
				const errorTitle = err?.message || ''
				if (errorTitle.includes('MaximumAttemptsExceededError')) {
					const errorMessage = err.messages?.[0] || err
					toast.error(__(errorMessage))
					setTimeout(() => {
						window.location.reload()
					}, 3000)
				}
			},
		}
	)
}

const resetQuiz = () => {
	activeQuestion.value = 0
	selectedOptions.value.splice(0, selectedOptions.value.length, ...[0, 0, 0, 0])
	showAnswers.length = 0
	possibleAnswer.value = null
	attemptedQuestions.value = []
	quizSubmission.reset()
	populateQuestions()
	setupTimer()
}

const getInstructions = (question) => {
	if (question.type == 'Choices')
		if (question.multiple) return __('Choose all answers that apply')
		else return __('Choose one answer')
	else return __('Type your answer')
}

const markLessonProgress = () => {
	let pathname = window.location.pathname.split('/')
	if (!pathname.includes('courses'))
		pathname = window.parent.location.pathname.split('/')
	if (pathname[2] != 'courses') return
	let lessonIndex = pathname.pop().split('-')

	if (lessonIndex.length == 2) {
		call('lms.lms.api.mark_lesson_progress', {
			course: pathname[3],
			chapter_number: lessonIndex[0],
			lesson_number: lessonIndex[1],
		})
	}
}

const handleSubmitClick = () => {
	if (!quiz.data.show_answers) {
		if (attemptedQuestions.value.length) {
			switchQuestion(activeQuestion.value)
		}
		showSubmissionConfirmation.value = true
	} else {
		submitQuiz()
	}
}

const paginationWindow = computed(() => {
	const total = questions.length
	const current = activeQuestion.value
	const pages = []
	const size = 5

	let start = Math.floor((current - 1) / size) * size + 1
	let end = Math.min(start + size - 1, total)

	if (start > 1) {
		pages.push('...')
	}

	for (let i = start; i <= end; i++) {
		pages.push(i)
	}

	if (end < total) {
		pages.push('...')
	}

	return pages
})

const markForReview = (event, questionNumber) => {
	if (event.target.checked) {
		if (!reviewQuestions.value.includes(questionNumber)) {
			reviewQuestions.value.push(questionNumber)
		}
	} else {
		reviewQuestions.value = reviewQuestions.value.filter(
			(num) => num !== questionNumber
		)
	}
}

const getSubmissionColumns = () => {
	return [
		{
			label: 'No.',
			key: 'idx',
		},
		{
			label: 'Date',
			key: 'creation',
		},
		{
			label: 'Score',
			key: 'score',
			align: 'center',
		},
		{
			label: 'Score out of',
			key: 'score_out_of',
			align: 'center',
		},
		{
			label: 'Percentage',
			key: 'percentage',
			align: 'center',
		},
	]
}

// ── NTA additions ────────────────────────────────────────────────────────────
const visitedQuestions = ref([])

watch(activeQuestion, (newVal, oldVal) => {
	if (oldVal > 0 && !visitedQuestions.value.includes(oldVal)) {
		visitedQuestions.value.push(oldVal)
	}
})

const saveAndNext = () => {
	const answers = getAnswers()
	if (answers.length) {
		if (!attemptedQuestions.value.includes(activeQuestion.value))
			attemptedQuestions.value.push(activeQuestion.value)
		addToLocalStorage()
	}
	if (activeQuestion.value < questions.length) {
		activeQuestion.value++
		selectedOptions.value.splice(0, 4, ...[0, 0, 0, 0])
		showAnswers.length = 0
		possibleAnswer.value = null
	}
}

const clearAnswer = () => {
	selectedOptions.value.splice(0, 4, ...[0, 0, 0, 0])
	possibleAnswer.value = null
	attemptedQuestions.value = attemptedQuestions.value.filter(q => q !== activeQuestion.value)
	let quizData = JSON.parse(localStorage.getItem(quiz.data.title))
	if (quizData) {
		quizData = quizData.filter(q => q.question_name !== currentQuestion.value)
		localStorage.setItem(quiz.data.title, JSON.stringify(quizData))
	}
}

const saveAndMarkForReview = () => {
	const answers = getAnswers()
	if (answers.length) {
		if (!attemptedQuestions.value.includes(activeQuestion.value))
			attemptedQuestions.value.push(activeQuestion.value)
		addToLocalStorage()
	}
	if (!reviewQuestions.value.includes(activeQuestion.value))
		reviewQuestions.value.push(activeQuestion.value)
	if (activeQuestion.value < questions.length) {
		activeQuestion.value++
		selectedOptions.value.splice(0, 4, ...[0, 0, 0, 0])
		showAnswers.length = 0
		possibleAnswer.value = null
	}
}

const markForReviewAndNext = () => {
	if (!reviewQuestions.value.includes(activeQuestion.value))
		reviewQuestions.value.push(activeQuestion.value)
	if (activeQuestion.value < questions.length) {
		activeQuestion.value++
		selectedOptions.value.splice(0, 4, ...[0, 0, 0, 0])
		showAnswers.length = 0
		possibleAnswer.value = null
	}
}

const getQuestionBtnClass = (n) => {
	const isActive = n === activeQuestion.value
	const isAnswered = attemptedQuestions.value.includes(n)
	const isMarked = reviewQuestions.value.includes(n)
	const isVisited = visitedQuestions.value.includes(n)
	if (isActive) return 'bg-blue-600 text-white ring-2 ring-blue-300'
	if (isAnswered && isMarked) return 'bg-purple-600 text-white'
	if (isMarked) return 'bg-purple-600 text-white'
	if (isAnswered) return 'bg-green-600 text-white'
	if (isVisited) return 'bg-red-500 text-white'
	return 'bg-gray-300 text-gray-700'
}

const notVisitedCount = computed(() =>
	questions.filter((_, i) => {
		const n = i + 1
		return n !== activeQuestion.value && !visitedQuestions.value.includes(n) && !attemptedQuestions.value.includes(n)
	}).length
)
const notAnsweredCount = computed(() =>
	visitedQuestions.value.filter(n => !attemptedQuestions.value.includes(n) && n !== activeQuestion.value).length
)
const answeredCount = computed(() =>
	attemptedQuestions.value.filter(n => !reviewQuestions.value.includes(n)).length
)
const markedOnlyCount = computed(() =>
	reviewQuestions.value.filter(n => !attemptedQuestions.value.includes(n)).length
)
const answeredAndMarkedCount = computed(() =>
	reviewQuestions.value.filter(n => attemptedQuestions.value.includes(n)).length
)
</script>
