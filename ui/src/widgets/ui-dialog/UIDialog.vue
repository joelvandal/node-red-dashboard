<template>
    <v-dialog v-model="show" persistent width="500">
        <v-card>
            <v-card-title>{{ title }}</v-card-title>
            <v-card-text>
                <div v-if="!props.raw">{{ props.body }}</div>
                <!-- eslint-disable-next-line vue/no-v-html -->
                <div v-else v-html="props.body" />
                <v-progress-linear v-if="props.progress" indeterminate :height="12" />
                <div v-if="props.showCountdown" class="nrdb-ui-notification-countdown">
                    <v-progress-linear v-model="countdown" :color="messages[id]?.color || (props.colorDefault ? 'primary' : color)" style="display: block; width: 100%" />
                </div>
            </v-card-text>
            <v-card-actions>
                <v-spacer />
                <v-btn v-if="props.show_cancel" variant="text" @click="close">{{ props.show_cancel_label || "Cancel" }}</v-btn>
                <v-btn v-for="(button, index) in props.actions" :key="index" variant="text" @click="buttonAction(button.id)">
                    {{ button.text }}
                </v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>
</template>

<script>
import { mapState } from 'vuex'

import { useDataTracker } from '../data-tracker.mjs' // eslint-disable-line import/order

export default {
    name: 'DBUIDialog',
    inject: ['$socket'],
    props: {
        id: { type: String, required: true },
        props: { type: Object, default: () => ({}) }
    },
    data () {
        return {
            show: false,
            tik: 0,
            countdown: 0,
            timeouts: {
                close: null,
                step: null
            }
        }
    },
    computed: {
        ...mapState('data', ['messages']),
        value: function () {
            return this.messages[this.id]?.payload
        },
        color: function () {
            if (this.messages[this.id]?.color) {
                return this.messages[this.id]?.color
            } else if (this.props.colorDefault) {
                return 'rgb(var(--v-theme-group-background))'
            } else {
                return this.props.color
            }
        }
    },
    created () {
        // can't do this in setup as we have custom onInput function
        useDataTracker(this.id, this.onMsgInput)
    },
    methods: {
        onMsgInput (msg) {
            this.$store.commit('data/bind', {
                widgetId: this.id,
                msg
            })

            this.show = this.props.show

            if (this.props.displayTime > 0) {
                // begin countdown
                this.startCountdown(this.props.displayTime * 1000)
            }
        },
        startCountdown (time) {
            this.tik = (new Date()).getTime()

            this.timeouts.close = setTimeout(() => {
                // close the notification after time has elapsed
                this.close()
            }, time)

            // update the progress bar every 100ms
            this.timeouts.step = setInterval(() => {
                const tok = (new Date()).getTime()
                // how many seconds have elapsed
                const elapsed = (tok - this.tik) / 1000
                // 100 = full bar, 0 = empty bar
                this.countdown = 100 - (elapsed / parseFloat(this.props.displayTime)) * 100
            }, 100)
        },
        close () {
            this.show = false

            clearTimeout(this.timeouts.close)
            clearInterval(this.timeouts.step)
            this.tik = null
            this.timeouts.close = null
            this.timeouts.step = null
        }
    }
}
</script>

<style scoped>
.nrdb-ui-notification {
    padding-top: 64px;
}

.nrdb-ui-notification .v-snackbar__wrapper {
    background-color: rgb(var(--v-theme-group-background));
    color: rgb(var(--v-theme-on-group-background));
    border-left: 6px solid var(--nrdb-ui-notification-color);
}
.nrdb-ui-notification .v-snackbar__content {
    padding-left: 8px;
}

.nrdb-ui-notification-countdown {
    position: absolute;
    width: calc(100% + 6px);
    top: 0;
    left: 0;
    margin-left: -6px;
    border-top-left-radius: 4px;
}

.nrdb-ui-notification h1,
.nrdb-ui-notification h2,
.nrdb-ui-notification h3,
.nrdb-ui-notification h4 {
    margin: 0.5rem 0;
}

</style>
