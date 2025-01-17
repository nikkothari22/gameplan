<template>
  <div class="flex select-none items-stretch space-x-1.5">
    <Popover>
      <template #target="{ togglePopover }">
        <button
          @click="togglePopover()"
          class="flex h-full items-center justify-center rounded-full border border-gray-300 px-2 py-1 transition hover:border-gray-400"
        >
          <ReactionFaceIcon />
        </button>
      </template>
      <template #body-main="{ togglePopover }">
        <div class="mt-1 inline-flex p-1">
          <div class="grid grid-cols-10 items-center space-x-0.5">
            <button
              class="h-6 w-6 rounded hover:bg-gray-50"
              v-for="emoji in standardEmojis"
              :key="emoji"
              @click="
                () => {
                  toggleReaction(emoji)
                  togglePopover()
                }
              "
              :disabled="
                $resources.addReaction.loading ||
                $resources.removeReaction.loading
              "
            >
              {{ emoji }}
            </button>
          </div>
        </div>
      </template>
    </Popover>
    <Transition
      enterActiveClass="transition duration-300 ease-out"
      enterFromClass="scale-75"
      enterToClass="scale-100"
      leaveActiveClass="transition duration-100 ease-in absolute"
      leaveFromClass="scale-100 opacity-100"
      leaveToClass="scale-90 opacity-0"
    >
      <TransitionGroup
        v-if="reactions.length"
        tag="div"
        class="flex items-stretch space-x-1.5"
        moveClass="transition duration-100 ease-in"
        enterActiveClass="transition duration-300 ease-out"
        enterFromClass="scale-75"
        enterToClass="scale-100"
        leaveActiveClass="transition duration-100 ease-in"
        leaveFromClass="scale-100 opacity-100"
        leaveToClass="scale-90 opacity-0"
      >
        <Tooltip v-for="(reactions, emoji) in reactionsCount" :key="emoji">
          <button
            class="flex items-center justify-center rounded-full border px-2 py-1 text-sm transition"
            :class="[
              reactions.userReacted
                ? 'border-blue-200 bg-blue-100 hover:border-blue-300'
                : 'border-gray-300 hover:border-gray-400',
            ]"
            @click="toggleReaction(emoji)"
          >
            {{ emoji }} {{ reactions.count }}
          </button>

          <template #body>
            <div
              class="max-w-[30ch] rounded-lg border border-gray-100 bg-gray-800 px-2 py-1 text-center text-xs text-white shadow-xl"
            >
              {{ toolTipText(reactions) }}
            </div>
          </template>
        </Tooltip>
      </TransitionGroup>
    </Transition>
  </div>
</template>
<script>
import { Popover, Tooltip } from 'frappe-ui'
import ReactionFaceIcon from './ReactionFaceIcon.vue'
import Avatar from 'frappe-ui/src/components/Avatar.vue'

export default {
  name: 'Reactions',
  props: ['reactions', 'doctype', 'name'],
  emits: ['update:reactions'],
  components: { ReactionFaceIcon, Popover, Avatar, Tooltip },
  resources: {
    addReaction() {
      return {
        method: 'frappe.client.insert',
        makeParams({ emoji, user }) {
          return {
            doc: {
              doctype: 'Team Reaction',
              parentfield: 'reactions',
              parenttype: this.doctype,
              parent: this.name,
              user,
              emoji,
            },
          }
        },
        onSuccess(data) {
          let reactions = data.reactions.map((d) => ({
            name: d.name,
            emoji: d.emoji,
            user: d.user,
          }))
          this.$emit('update:reactions', reactions)
        },
      }
    },
    removeReaction() {
      return {
        method: 'frappe.client.delete',
        makeParams(name) {
          return {
            doctype: 'Team Reaction',
            name,
          }
        },
      }
    },
  },
  methods: {
    toggleReaction(emoji) {
      if (this.$readOnlyMode) return;
      let existingReaction = this.reactions.find(
        (r) => r.user === this.$user().name && r.emoji === emoji
      )
      if (existingReaction) {
        this.removeReaction(existingReaction)
      } else {
        this.addReaction(emoji)
      }
    },
    addReaction(emoji) {
      const user = this.$user().name
      let reactions = [
        ...this.reactions,
        {
          emoji,
          user,
          name: `new-emoji-${this.reactions.length}`,
        },
      ]
      this.$emit('update:reactions', reactions)
      this.$resources.addReaction.submit({ emoji, user })
    },
    removeReaction(reaction) {
      // update local
      let reactions = this.reactions.filter((r) => r !== reaction)
      this.$emit('update:reactions', reactions)

      // update server
      this.$resources.removeReaction.submit(reaction.name)
    },
    toolTipText(reactions) {
      return reactions.users
        .map((user) => {
          if (user) {
            return this.$user(user).full_name.trim()
          }
          return ''
        })
        .join(', ')
    },
  },
  computed: {
    reactionsCount() {
      let out = {}
      for (let reaction of this.reactions) {
        if (!out[reaction.emoji]) {
          out[reaction.emoji] = { count: 0, users: [], userReacted: false }
        }
        out[reaction.emoji].count++
        out[reaction.emoji].users.push(reaction.user)
        if (reaction.user === this.$user().name) {
          out[reaction.emoji].userReacted = true
        }
      }
      return out
    },
    standardEmojis() {
      return [
        '👍',
        '👎',
        '💖',
        '🔥',
        '👏🏻',
        '🤔',
        '😱',
        '🤯',
        '😡',
        '⚡️',
        '🥳',
        '🎉',
        '💩',
        '🤩',
        '😢',
        '😂',
        '🍿',
        '🙈',
        '🌚',
        '🚀',
      ]
    },
  },
}
</script>
