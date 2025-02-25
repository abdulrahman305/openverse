<script setup lang="ts">
import { useI18n, useNuxtApp } from "#imports"
import { computed, nextTick, onMounted, ref } from "vue"

import { useResizeObserver, watchDebounced } from "@vueuse/core"

import type { SupportedMediaType } from "#shared/constants/media"
import type { Tag } from "#shared/types/media"
import { focusElement } from "~/utils/focus-management"
import { useSearchStore } from "~/stores/search"

import VTag from "~/components/VTag/VTag.vue"
import VButton from "~/components/VButton.vue"
import VIcon from "~/components/VIcon/VIcon.vue"

// The number of rows to display before collapsing the tags
const ROWS_TO_DISPLAY = 3

const props = defineProps<{
  tags: Tag[]
  mediaType: SupportedMediaType
}>()

const tagsContainerRef = ref<HTMLElement>()

const searchStore = useSearchStore()
const { $sendCustomEvent } = useNuxtApp()

const localizedTagPath = (tag: string) => {
  return searchStore.getCollectionPath({
    type: props.mediaType,
    collectionParams: { collection: "tag", tag },
  })
}

const normalizedTags = computed(() => {
  return Array.from(new Set(props.tags.map((tag) => tag.name)))
})

const collapsibleRowsStartAt = ref<number>()
const { localeProperties } = useI18n({ useScope: "global" })
const dir = computed(() => {
  return localeProperties.value.dir
})

function isFurtherInline(previous: HTMLElement, current: HTMLElement) {
  if (dir.value === "rtl") {
    return previous.offsetLeft < current.offsetLeft
  }
  return previous.offsetLeft > current.offsetLeft
}

function findRowStartsAt(parent: HTMLElement) {
  const children = Array.from(parent.children)
  if (!children.length) {
    return 0
  }
  let rowCount = 0
  for (let i = 0; i < children.length; i++) {
    const child = children[i] as HTMLElement
    const previous = child.previousElementSibling as HTMLElement
    if (!previous) {
      rowCount++
    } else if (isFurtherInline(previous, child)) {
      rowCount++
    }
    if (rowCount === ROWS_TO_DISPLAY + 1) {
      return i
    }
  }
  return children.length
}

/**
 * Only the first 3 rows of tags are visible by default.
 * If we hide the tags using CSS only, they will be tabbable,
 * even though they are not visible.
 */
const visibleTags = computed<string[]>(() => {
  return collapsibleRowsStartAt.value && buttonStatus.value === "show"
    ? normalizedTags.value.slice(0, collapsibleRowsStartAt.value)
    : normalizedTags.value
})

const hasOverflow = computed(() => {
  return (
    collapsibleRowsStartAt.value &&
    collapsibleRowsStartAt.value < normalizedTags.value.length
  )
})

onMounted(() => {
  /**
   * Find the index of the first item after the third row of tags. This is used
   * to determine which tags to hide.
   */
  if (tagsContainerRef.value) {
    collapsibleRowsStartAt.value = findRowStartsAt(tagsContainerRef.value)
  }
})

const buttonStatus = ref<"show" | "hide">("show")
/**
 * Toggles the text for the "Show more" button. When showing more tags, we also
 * focus the first tag in the newly-opened row for a11y.
 */
const handleClick = () => {
  buttonStatus.value = buttonStatus.value === "show" ? "hide" : "show"
  $sendCustomEvent("TOGGLE_TAG_EXPANSION", {
    toState: buttonStatus.value === "show" ? "collapsed" : "expanded",
  })
  if (buttonStatus.value === "hide" && collapsibleRowsStartAt.value) {
    nextTick(() => {
      if (!collapsibleRowsStartAt.value) {
        return
      }
      const firstTagInFourthRow = tagsContainerRef.value?.children.item(
        collapsibleRowsStartAt.value
      ) as HTMLElement
      focusElement(firstTagInFourthRow?.querySelector("a"))
    })
  }
}

const heightClass = computed(() => {
  if (!hasOverflow.value) {
    return "max-h-none"
  }
  /**
   * Height is 3 rows of tags, gaps, and a padding for the focus rings.
   * 3 * 2rem (tags) + 2 * 0.75rem (2 gaps) + 0.1875rem (margin for the focus ring)
   */
  return buttonStatus.value === "show" ? "max-h-[7.6875rem]" : "mah-h-none"
})

const listWidth = ref<number>()
useResizeObserver(tagsContainerRef, (entries) => {
  listWidth.value = entries[0].contentRect.width
})

watchDebounced(
  listWidth,
  (newWidth, oldWidth) => {
    if (!tagsContainerRef.value) {
      return
    }
    const isWidening = oldWidth && newWidth && newWidth > oldWidth

    if (isWidening) {
      collapsibleRowsStartAt.value = normalizedTags.value.length
    }
    nextTick(() => {
      if (tagsContainerRef.value) {
        collapsibleRowsStartAt.value = findRowStartsAt(tagsContainerRef.value)
      }
    })
  },
  { debounce: 300 }
)
</script>

<template>
  <div class="-my-1.5px">
    <ul
      ref="tagsContainerRef"
      class="flex flex-wrap gap-3 overflow-y-hidden p-1.5px"
      :class="heightClass"
    >
      <li v-for="tag in visibleTags" :key="tag">
        <VTag :href="localizedTagPath(tag)">{{ tag }}</VTag>
      </li>
    </ul>
    <VButton
      v-if="hasOverflow"
      size="small"
      variant="transparent-tx"
      has-icon-end
      class="label-bold -ms-2 mt-4 hover:underline"
      :aria-expanded="buttonStatus === 'show' ? 'false' : 'true'"
      @click="handleClick"
      >{{
        $t(
          buttonStatus === "show"
            ? "mediaDetails.tags.showMore"
            : "mediaDetails.tags.showLess"
        )
      }}<VIcon
        name="caret-down"
        :size="4"
        :class="{ '-scale-y-100 transform': buttonStatus === 'hide' }"
    /></VButton>
  </div>
</template>
