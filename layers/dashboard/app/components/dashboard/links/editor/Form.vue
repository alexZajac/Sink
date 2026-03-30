<script setup lang="ts">
import type { AnyFieldApi, Link, LinkFormData, LinkTarget } from '@/types'
import { LinkSchema, nanoid } from '#shared/schemas/link'
import { useForm } from '@tanstack/vue-form'
import { PlusIcon, Shuffle, Sparkles, Trash2Icon } from 'lucide-vue-next'
import { toast } from 'vue-sonner'
import { z } from 'zod'

const props = defineProps<{
  link: Partial<Link>
  isEdit: boolean
}>()

const emit = defineEmits<{
  success: [link: Link]
}>()

const { t } = useI18n()

const urlValidator = LinkSchema.shape.url
const slugValidator = LinkSchema.shape.slug
const commentValidator = z.string().max(500).optional()
const optionalUrlValidator = z.string().trim().url().max(2048).optional().or(z.literal(''))

const generateSlug = nanoid()

// When editing a weighted link, targets[0] is the primary URL's weighted entry.
// Separate it out so the form only manages the variant targets.
const storedTargets = props.link.targets ?? []
const initialPrimaryWeight = storedTargets[0]?.weight
const initialFormTargets = storedTargets.length >= 1 ? storedTargets.slice(1) : []

const primaryWeight = ref<number | undefined>(initialPrimaryWeight)

const form = useForm({
  defaultValues: {
    url: props.link.url ?? '',
    slug: props.link.slug ?? '',
    comment: props.link.comment ?? '',
    expiration: props.link.expiration
      ? unix2date(props.link.expiration)
      : undefined,
    google: props.link.google ?? '',
    apple: props.link.apple ?? '',
    title: props.link.title ?? '',
    description: props.link.description ?? '',
    image: props.link.image ?? '',
    cloaking: props.link.cloaking ?? false,
    redirectWithQuery: props.link.redirectWithQuery ?? false,
    password: props.link.password ?? '',
    unsafe: props.link.unsafe ?? false,
    targets: initialFormTargets,
  } satisfies LinkFormData,
  onSubmit: async ({ value }) => {
    try {
      const hasWeightedTargets = value.targets && value.targets.length >= 1
      const linkData = {
        url: value.url,
        slug: value.slug,
        comment: value.comment || undefined,
        expiration: value.expiration
          ? date2unix(value.expiration, 'end')
          : undefined,
        google: value.google || undefined,
        apple: value.apple || undefined,
        title: value.title || undefined,
        description: value.description || undefined,
        image: value.image || undefined,
        cloaking: value.cloaking,
        redirectWithQuery: value.redirectWithQuery,
        password: value.password || undefined,
        unsafe: value.unsafe || undefined,
        targets: hasWeightedTargets
          ? [{ url: value.url, weight: primaryWeight.value ?? 0 }, ...value.targets!]
          : undefined,
      }
      const { link: newLink } = await useAPI<{ link: Link }>(
        props.isEdit ? '/api/link/edit' : '/api/link/create',
        {
          method: props.isEdit ? 'PUT' : 'POST',
          body: linkData,
        },
      )
      emit('success', newLink)
      toast(props.isEdit ? t('links.update_success') : t('links.create_success'))
    }
    catch (error) {
      console.error(error)
      toast.error(props.isEdit ? t('links.update_failed') : t('links.create_failed'), {
        description: error instanceof Error ? error.message : String(error),
      })
    }
  },
})

function makeValidator<T>(schema: z.ZodSchema<T>) {
  return ({ value }: { value: T }) => {
    const result = schema.safeParse(value)
    return result.success ? undefined : result.error.errors[0]?.message
  }
}

const validateUrl = makeValidator(urlValidator)
const validateSlug = makeValidator(slugValidator)
const validateComment = makeValidator(commentValidator)
const validateOptionalUrl = makeValidator(optionalUrlValidator)

function isInvalid(field: AnyFieldApi) {
  return field.state.meta.isTouched && !field.state.meta.isValid
}

function getAriaInvalid(field: AnyFieldApi) {
  return isInvalid(field) ? 'true' : undefined
}

function formatErrors(errors: unknown[]): string[] {
  return errors
    .map((e) => {
      if (typeof e === 'string')
        return e
      if (e && typeof e === 'object' && 'message' in e && typeof e.message === 'string')
        return e.message
      return null
    })
    .filter((m): m is string => m !== null)
}

function randomSlug() {
  form.setFieldValue('slug', generateSlug())
}

const aiSlugPending = ref(false)
async function aiSlug() {
  const url = form.getFieldValue('url')
  if (!url)
    return

  aiSlugPending.value = true
  try {
    const result = await useAPI<{ slug: string }>('/api/link/ai', {
      query: { url },
    })
    form.setFieldValue('slug', result.slug)
  }
  catch (error) {
    console.error(error)
    toast.error(t('links.ai_slug_failed'), {
      description: error instanceof Error ? error.message : String(error),
    })
  }
  finally {
    aiSlugPending.value = false
  }
}

const currentSlug = form.useStore(state => state.values.slug || '')
const targetsValue = form.useStore(state => state.values.targets ?? [])

const requestUrl = useRequestURL()
const shortLinkPreview = computed(() =>
  currentSlug.value ? `${requestUrl.origin}/${currentSlug.value}` : '',
)

function setTargets(newTargets: LinkTarget[]) {
  form.setFieldValue('targets', newTargets)
}

function addTarget() {
  if (targetsValue.value.length === 0) {
    primaryWeight.value = 0.5
    setTargets([{ url: '', weight: 0.5 }])
  }
  else {
    setTargets([...targetsValue.value, { url: '', weight: 0 }])
  }
}

function removeTarget(index: number) {
  const next = targetsValue.value.filter((_, i) => i !== index)
  if (next.length === 0)
    primaryWeight.value = undefined
  setTargets(next)
}

function updateTargetUrl(index: number, url: string) {
  setTargets(targetsValue.value.map((t, i) => i === index ? { ...t, url } : t))
}

function updateTargetWeight(index: number, raw: string) {
  const weight = Number.parseFloat(raw)
  setTargets(targetsValue.value.map((t, i) => i === index ? { ...t, weight: Number.isNaN(weight) ? 0 : weight } : t))
}

const weightSum = computed(() =>
  (primaryWeight.value ?? 0) + targetsValue.value.reduce((s, t) => s + (t.weight || 0), 0),
)
const weightsValid = computed(() =>
  targetsValue.value.length === 0 || Math.abs(weightSum.value - 1.0) < 0.001,
)

const { previewMode } = useRuntimeConfig().public

defineExpose({ randomSlug })
</script>

<template>
  <form
    id="link-editor-form"
    class="w-full space-y-4 px-1"
    @submit.prevent="form.handleSubmit"
  >
    <p
      v-if="previewMode"
      class="text-sm text-muted-foreground"
    >
      {{ $t('links.preview_mode_tip') }}
    </p>

    <FieldGroup>
      <form.Field
        v-slot="{ field }"
        name="url"
        :validators="{ onBlur: validateUrl }"
      >
        <Field :data-invalid="isInvalid(field)">
          <FieldLabel :for="field.name">
            {{ $t('links.form.url') }}
          </FieldLabel>
          <div class="flex items-center gap-2">
            <Input
              :id="field.name"
              :name="field.name"
              :model-value="field.state.value"
              :aria-invalid="getAriaInvalid(field)"
              placeholder="https://example.com"
              autocomplete="url"
              class="flex-1"
              @blur="field.handleBlur"
              @input="field.handleChange(($event.target as HTMLInputElement).value)"
            />
            <Input
              v-if="targetsValue.length > 0"
              type="number"
              :model-value="primaryWeight"
              placeholder="0.5"
              step="0.01"
              min="0.01"
              max="1"
              class="w-20 shrink-0"
              aria-label="Primary URL weight"
              @input="primaryWeight = Number.parseFloat(($event.target as HTMLInputElement).value)"
            />
          </div>
          <FieldError
            v-if="isInvalid(field)"
            :errors="formatErrors(field.state.meta.errors)"
          />
        </Field>
      </form.Field>

      <!-- Weighted variant rows -->
      <div v-if="targetsValue.length > 0" class="space-y-2">
        <div
          v-for="(target, index) in targetsValue"
          :key="index"
          class="flex items-center gap-2"
        >
          <Input
            :model-value="target.url"
            placeholder="https://example.com"
            autocomplete="off"
            class="flex-1"
            @input="updateTargetUrl(index, ($event.target as HTMLInputElement).value)"
          />
          <Input
            type="number"
            :model-value="target.weight || ''"
            placeholder="0.5"
            step="0.01"
            min="0.01"
            max="1"
            class="w-20 shrink-0"
            @input="updateTargetWeight(index, ($event.target as HTMLInputElement).value)"
          />
          <Button
            type="button"
            variant="ghost"
            size="icon"
            class="shrink-0"
            aria-label="Remove target"
            @click="removeTarget(index)"
          >
            <Trash2Icon class="h-4 w-4" />
          </Button>
        </div>
        <p v-if="!weightsValid" class="text-xs text-destructive">
          {{ $t('links.form.weighted_targets_sum_warning', { sum: weightSum.toFixed(3) }) }}
        </p>
      </div>

      <Button
        v-if="targetsValue.length < 20"
        type="button"
        variant="ghost"
        size="sm"
        class="
          -mt-1 h-auto w-full justify-start p-0 text-xs text-muted-foreground
          hover:text-foreground
        "
        @click="addTarget"
      >
        <PlusIcon class="mr-1 h-3 w-3" />
        {{ $t('links.form.weighted_target_add') }}
      </Button>

      <form.Field
        v-slot="{ field }"
        name="slug"
        :validators="{ onBlur: validateSlug }"
      >
        <Field :data-invalid="isInvalid(field)">
          <div class="flex items-center justify-between">
            <FieldLabel :for="field.name">
              {{ $t('links.form.slug') }}
            </FieldLabel>
            <div v-if="!isEdit" class="flex space-x-3">
              <Button
                variant="ghost"
                size="icon"
                class="h-auto w-auto p-0"
                aria-label="Generate random slug"
                @click="randomSlug"
              >
                <Shuffle class="h-4 w-4" />
              </Button>
              <Button
                variant="ghost"
                size="icon"
                class="h-auto w-auto p-0"
                aria-label="Generate AI slug"
                :disabled="aiSlugPending"
                @click="aiSlug"
              >
                <Sparkles
                  class="h-4 w-4"
                  :class="{ 'animate-bounce': aiSlugPending }"
                />
              </Button>
            </div>
          </div>
          <Input
            :id="field.name"
            :name="field.name"
            :model-value="field.state.value"
            :disabled="isEdit"
            :aria-invalid="getAriaInvalid(field)"
            placeholder="my-short-link"
            autocomplete="off"
            @blur="field.handleBlur"
            @input="field.handleChange(($event.target as HTMLInputElement).value)"
          />
          <FieldError
            v-if="isInvalid(field)"
            :errors="formatErrors(field.state.meta.errors)"
          />
          <p
            v-if="shortLinkPreview" class="
              truncate text-xs text-muted-foreground
            "
          >
            {{ shortLinkPreview }}
          </p>
        </Field>
      </form.Field>

      <form.Field
        v-slot="{ field }"
        name="comment"
        :validators="{ onBlur: validateComment }"
      >
        <Field :data-invalid="isInvalid(field)">
          <FieldLabel :for="field.name">
            {{ $t('links.form.comment') }}
          </FieldLabel>
          <Textarea
            :id="field.name"
            :name="field.name"
            :model-value="field.state.value"
            :aria-invalid="getAriaInvalid(field)"
            @blur="field.handleBlur"
            @input="field.handleChange(($event.target as HTMLTextAreaElement).value)"
          />
          <FieldError
            v-if="isInvalid(field)"
            :errors="formatErrors(field.state.meta.errors)"
          />
        </Field>
      </form.Field>
    </FieldGroup>

    <DashboardLinksEditorAdvanced
      :form="form"
      :validate-optional-url="validateOptionalUrl"
      :is-invalid="isInvalid"
      :get-aria-invalid="getAriaInvalid"
      :format-errors="formatErrors"
      :current-slug="currentSlug"
    />
  </form>
</template>
