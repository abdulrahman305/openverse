<script setup lang="ts">
/**
 * Displays the waveform and basic information about the track, along with
 * controls to play, pause or seek to a point on the track.
 */
import { useI18n, useNuxtApp } from "#imports"
import { computed, ref, watch } from "vue"

import type { AudioStatus } from "#shared/constants/audio"
import type { AudioInteraction } from "#shared/types/analytics"
import type { AudioDetail } from "#shared/types/media"
import { useActiveMediaStore } from "~/stores/active-media"
import { useMediaStore } from "~/stores/media"
import { useActiveAudio } from "~/composables/use-active-audio"
import { defaultRef } from "~/composables/default-ref"

import VAudioControl from "~/components/VAudioTrack/VAudioControl.vue"
import VWaveform from "~/components/VAudioTrack/VWaveform.vue"
import VGlobalLayout from "~/components/VAudioTrack/layouts/VGlobalLayout.vue"

const props = defineProps<{
  /**
   * The information about the track, typically from a track's detail endpoint
   */
  audio: AudioDetail
}>()

const { t } = useI18n({ useScope: "global" })
const activeMediaStore = useActiveMediaStore()
const activeAudio = useActiveAudio()
const { $sendCustomEvent } = useNuxtApp()

const ariaLabel = computed(() =>
  t("audioTrack.ariaLabel", { title: props.audio.title })
)

const status = ref<AudioStatus>("paused")
const currentTime = ref(0)
const duration = defaultRef(() => {
  if (typeof props.audio?.duration === "number") {
    return props.audio.duration / 1e3
  }
  return 0
})

const setPlaying = () => {
  if (props.audio.hasLoaded) {
    status.value = "playing"
  } else {
    status.value = "loading"
  }
  updateTimeLoop()
}
const setPaused = () => (status.value = "paused")
const setPlayed = () => (status.value = "played")

const setTimeWhenPaused = (event: Event) => {
  if (status.value !== "playing" && event.target) {
    currentTime.value = (event.target as HTMLAudioElement).currentTime ?? 0
    if (status.value === "played") {
      // Set to pause to remove replay icon
      status.value = "paused"
    }
  }
}
const setDuration = () => {
  if (activeAudio.obj.value) {
    duration.value = activeAudio.obj.value.duration
  }
}

const updateTimeLoop = () => {
  if (
    activeAudio.obj.value &&
    (status.value === "playing" || status.value === "loading")
  ) {
    currentTime.value = activeAudio.obj.value.currentTime
    window.requestAnimationFrame(updateTimeLoop)
  }
}

const mediaStore = useMediaStore()
const setLoaded = () => {
  mediaStore.setMediaProperties("audio", props.audio.id, {
    hasLoaded: true,
  })
  status.value = "playing"
}
const setWaiting = () => {
  status.value = "loading"
}

const eventMap = {
  play: setPlaying,
  pause: setPaused,
  ended: setPlayed,
  timeupdate: setTimeWhenPaused,
  durationchange: setDuration,
  waiting: setWaiting,
  playing: setLoaded,
} as const

watch(
  activeAudio.obj,
  (audio, _, onInvalidate) => {
    if (!audio) {
      return
    }

    Object.entries(eventMap).forEach(([name, fn]) =>
      audio.addEventListener(name, fn)
    )

    currentTime.value = audio.currentTime
    if (audio.duration && !isNaN(audio.duration)) {
      duration.value = audio.duration
    }

    /**
     * By the time the `activeAudio` is updated and a rerender
     * happens (triggering this watch function), all the events
     * we've registered above will already have fired, so we
     * need to derive the current status of the audio from the
     * `paused` and `ended` booleans on the audio object.
     *
     * In practice this will always result in the status being
     * set to `playing` as the active audio is only updated when
     * a new track is set to play. But for good measure we might
     * as well do this robustly and make sure that the status is
     * always synced any time the active audio hangs.
     */
    if (audio.paused) {
      if (audio.ended) {
        setPlayed()
      } else {
        setPaused()
      }
    } else {
      setPlaying()
    }

    onInvalidate(() => {
      Object.entries(eventMap).forEach(([name, fn]) =>
        audio.removeEventListener(name, fn)
      )
    })
  },
  { immediate: true }
)

const play = () => activeMediaStore.playAudio(activeAudio.obj.value)
const pause = () => activeAudio.obj.value?.pause()

/* Timekeeping */
const message = computed(() =>
  activeMediaStore.message
    ? t(`audioTrack.messages.${activeMediaStore.message}`)
    : ""
)

/* Interface with VAudioControl */

const sendAudioInteractionEvent = (event: AudioInteraction) => {
  $sendCustomEvent("AUDIO_INTERACTION", {
    id: props.audio.id,
    provider: props.audio.provider,
    event,
    component: "VGlobalAudioTrack",
  })
}
const handleToggle = (state?: Exclude<AudioStatus, "loading" | "played">) => {
  if (!state) {
    switch (status.value) {
      case "playing": {
        state = "paused"
        break
      }
      case "paused":
      case "played": {
        state = "playing"
        break
      }
    }
  }
  let event: AudioInteraction | undefined = undefined
  switch (state) {
    case "playing": {
      play()
      event = "play"
      break
    }
    case "paused": {
      pause()
      event = "pause"
      break
    }
  }
  if (event) {
    sendAudioInteractionEvent(event)
  }
}

/* Interface with VWaveform */

const handleSeeked = (frac: number) => {
  if (activeAudio.obj.value) {
    activeAudio.obj.value.currentTime = frac * duration.value
    sendAudioInteractionEvent("seek")
  }
}
</script>

<template>
  <div
    class="audio-track relative rounded"
    :aria-label="ariaLabel"
    role="region"
  >
    <VGlobalLayout :audio="audio">
      <template #controller="waveformProps">
        <VWaveform
          v-bind="waveformProps"
          :peaks="audio.peaks"
          :audio-id="audio.id"
          :current-time="currentTime"
          :duration="duration"
          :message="message"
          @seeked="handleSeeked"
          @toggle-playback="handleToggle"
        />
      </template>

      <template #audio-control="audioControlProps">
        <VAudioControl
          v-bind="audioControlProps"
          :status="status"
          @toggle="handleToggle"
        />
      </template>
    </VGlobalLayout>
  </div>
</template>
