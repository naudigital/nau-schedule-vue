<script setup lang="ts">
import ModalTemplate from './ModalTemplate.vue'
import {
  type LessonFormProps,
  type Lesson,
  PlaceType,
} from '@/utils/types.d.js'
import { pad, placeType } from '@/utils/functions.js'
import { useScheduleStore } from '@/stores/schedule'
import { ref, watch } from 'vue'
import { useGroupsStore } from '@/stores/groups'
const scheduleStore = useScheduleStore()
const groupsStore = useGroupsStore()

const props = defineProps<{ state: LessonFormProps | undefined }>()
const emit = defineEmits<{
  (eventName: 'close', eventValue: void): void
  (eventName: 'submit', eventValue: Lesson): void
}>()

const currentDate = new Date()
const lesson = ref<Lesson>({
  day_number: 1,
  week_number: 1,
  template: null,
  lecturers: undefined,
  subgroup: 0,
  comment: undefined,
  names: [''],
  time: 480,
  duration: 95,
  places: [],
  lesson_type: 'lecture',
  recordings: [],
  start_date: [
    props.state?.year ?? currentDate.getFullYear(),
    props.state?.month ?? currentDate.getMonth() + 1,
    props.state?.day ?? currentDate.getDate(),
  ],
  end_date: [
    props.state?.year ?? currentDate.getFullYear(),
    props.state?.month ?? currentDate.getMonth() + 1,
    props.state?.day ?? currentDate.getDate(),
  ],
})

const applyTemplate = (e: Event) => {
  const target = e.target as HTMLSelectElement
  const templates = scheduleStore.getTemplates(
    props.state?.group_code as string,
  )
  const template = templates.find((t) => t.id === target.value)
  if (!template) {
    lesson.value.template = null
    return
  }
  lesson.value.template = template.id
  Object.entries(template).forEach(([key, value]) => {
    if (typeof value !== 'undefined') {
      lesson.value = {
        ...lesson.value,
        [key as keyof Lesson]: value,
      }
    }
  })
}

watch(
  () => props.state?.id,
  async (code) => {
    if (!code || !props.state?.group_code) return
    lesson.value = await scheduleStore.getLesson(props.state?.group_code, code)
  },
  { immediate: true },
)
scheduleStore.loadLecturers()
</script>
<template>
  <ModalTemplate :state="!!state" @close="emit('close')">
    <template v-slot:headerLeft>
      <span>Пара</span>
    </template>
    <template v-slot:headerCenter>
      <!-- <span>Lesson</span> -->
    </template>
    <div>
      <form @submit.prevent="emit('submit', lesson)" style="padding: 12px">
        <p>Пресети</p>
        <select
          v-if="state?.group_code && lesson?.template"
          :value="lesson?.template"
          @change="applyTemplate($event)"
        >
          <option :value="null">Нічого</option>
          <option
            v-for="(preset, i) in scheduleStore.getTemplates(state.group_code)"
            :key="i"
            :value="preset.id"
          >
            {{ `${preset.id} - ${preset.names[0]}` }}
          </option>
        </select>
        <p>Назва пари</p>
        <div v-if="Array.isArray(lesson.names)">
          <div v-for="(name, i) in lesson.names" :key="i">
            {{ i + 1 }}.
            <input
              v-model="lesson.names[i]"
              :key="i"
              type="text"
              :placeholder="i === 0 ? 'Українською' : 'Англійською'"
              name="lesson-name"
              id="lesson-form__lesson-name"
            />
            <button
              v-if="lesson.names.length > 1"
              type="button"
              @click="lesson.names.splice(i, 1)"
            >
              X
            </button>
          </div>
        </div>
        <button
          v-if="!Array.isArray(lesson.names) || lesson.names.length < 2"
          type="button"
          @click="
            Array.isArray(lesson.names)
              ? lesson.names.push('')
              : (lesson.names = [''])
          "
        >
          + Назва
        </button>
        <p>Викладач</p>
        <div v-if="Array.isArray(lesson.lecturers)">
          <div v-for="(code, index) in lesson.lecturers" :key="index">
            <select
              :value="code"
              @change="
                lesson.lecturers[index] = (
                  $event?.target as HTMLSelectElement
                ).value
              "
            >
              <option :value="null">Ніхто</option>
              <option
                v-for="lecturer in scheduleStore.lecturers"
                :key="lecturer.code"
                :value="lecturer.code"
              >
                {{
                  `${lecturer.surname} ${lecturer.name} ${lecturer.patronymic}`
                }}
              </option>
            </select>
            <button
              v-if="lesson?.lecturers && lesson.lecturers.length > 1"
              type="button"
              @click="lesson.lecturers.splice(index, 1)"
            >
              X
            </button>
          </div>
        </div>
        <button
          v-if="!Array.isArray(lesson.lecturers) || lesson.lecturers.length < 2"
          type="button"
          @click="
            Array.isArray(lesson.lecturers)
              ? lesson.lecturers.push('')
              : (lesson.lecturers = [''])
          "
        >
          + Викладач
        </button>
        <p>День</p>
        <select v-model="lesson.day_number">
          <option value="1">Понеділок</option>
          <option value="2">Вівторок</option>
          <option value="3">Середа</option>
          <option value="4">Четвер</option>
          <option value="5">П'ятниця</option>
          <option value="6">Субота</option>
          <option value="7">Неділя</option>
        </select>
        <p>Тиждень</p>
        <select v-model="lesson.week_number">
          <option value="1">Перший</option>
          <option value="2">Другий</option>
        </select>
        <p>Підгрупа</p>
        <select v-model="lesson.subgroup">
          <option :value="0">Обидві</option>
          <option
            v-if="
              groupsStore.groups.find((g) => g.code === state?.group_code)
                ?.has_second_subgroup
            "
            :value="1"
          >
            Перша
          </option>
          <option
            v-if="
              groupsStore.groups.find((g) => g.code === state?.group_code)
                ?.has_second_subgroup
            "
            :value="2"
          >
            Друга
          </option>
        </select>
        <p>Тип</p>
        <select v-model="lesson.lesson_type">
          <option value="lecture">Лекція</option>
          <option value="practical">Практика</option>
          <option value="lab">Лабораторна</option>
        </select>
        <p>Час</p>
        <input
          :value="`${pad(Math.floor((lesson.time ?? 0) / 60), 2)}:${pad(
            (lesson.time ?? 0) % 60,
            2,
          )}`"
          type="time"
          name="lesson-time"
          id="lesson-form__lesson-time"
          @change="
            lesson.time = ($event.target as HTMLInputElement).value
              .split(':')
              .map((v) => parseInt(v))
              .reduce((acc, cur, i) => acc + cur * (i === 0 ? 60 : 1), 0)
          "
        />
        <p>Тривалість</p>
        <input
          :value="`${pad(Math.floor((lesson.duration ?? 0) / 60), 2)}:${pad(
            (lesson.duration ?? 0) % 60,
            2,
          )}`"
          type="time"
          name="lesson-duration"
          id="lesson-form__lesson-duration"
          @change="
            lesson.duration = ($event.target as HTMLInputElement).value
              .split(':')
              .map((v) => parseInt(v))
              .reduce((acc, cur, i) => acc + cur * (i === 0 ? 60 : 1), 0)
          "
        />
        <p>Початок</p>
        <input
          :value="
            lesson.start_date
              .map((e, i) => (i !== 0 ? pad(e, 2) : e.toString()))
              .join('-')
          "
          type="date"
          name="lesson-start-date"
          id="lesson-form__lesson-start-date"
          @change="
            lesson.start_date = ($event.target as HTMLInputElement).value
              .split('-')
              .map((v) => parseInt(v))
          "
        />
        <p>Кінець</p>
        <input
          :value="
            lesson.end_date
              .map((e, i) => (i !== 0 ? pad(e, 2) : e.toString()))
              .join('-')
          "
          type="date"
          name="lesson-end-date"
          id="lesson-form__lesson-end-date"
          :min="
            lesson.start_date
              .map((e, i) => (i !== 0 ? pad(e, 2) : e.toString()))
              .join('-')
          "
          @change="
            lesson.end_date = ($event.target as HTMLInputElement).value
              .split('-')
              .map((v) => parseInt(v))
          "
        />
        <p>Місце</p>
        <div v-if="Array.isArray(lesson.places)">
          <div v-for="(name, i) in lesson.places" :key="`${name}-${i}`">
            {{ i + 1 }}.
            <input
              v-model="lesson.places[i].text"
              type="text"
              name="lesson-name"
              id="lesson-form__lesson-name"
            />
            <select v-model="lesson.places[i].place_type">
              <option
                v-for="place_type in Object.entries(placeType)"
                :key="place_type[1]"
                :value="place_type[1]"
              >
                {{ place_type[0] }}
              </option>
            </select>
            <button
              v-if="lesson.places?.length > 1"
              type="button"
              @click="lesson.places.splice(i, 1)"
            >
              X
            </button>
          </div>
        </div>
        <button
          v-if="!Array.isArray(lesson.places) || lesson.places.length < 5"
          type="button"
          @click="
            Array.isArray(lesson.places)
              ? lesson.places.push({
                  text: '',
                  place_type: PlaceType.offline_classroom,
                })
              : (lesson.places = [
                  { text: '', place_type: PlaceType.offline_classroom },
                ])
          "
        >
          + Місце
        </button>
        <button type="submit">
          {{ lesson.code ? 'Зберегти' : 'Створити' }}
        </button>
        {{ lesson }}
      </form>
    </div>
  </ModalTemplate>
</template>
<style scoped>
form * {
  max-width: 100%;
}
form button[type='submit'] {
  width: 100%;
  margin-top: 16px;
}
</style>