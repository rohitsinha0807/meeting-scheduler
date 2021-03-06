<script>
/* global grecaptcha */
import moment from 'moment-timezone';
import InfiniteLoading from 'vue-infinite-loading';
import { MAX_TIME_SLOTS_PER_SCROLL, EVENTS } from 'constants';
import { mapState } from 'vuex';
import date from '../utils/date';
import TextInput from './common/TextInput';
import Textarea from './common/Textarea';
import InputLabel from './common/InputLabel';
import Title from './common/Title';
import Button from './common/Button';
import TimeSlotBlock from './TimeSlotBlock';


let onRecaptchaScriptLoad;
/**
* Load reCaptcha script
*
* @returns {Promise}
*/
function loadRecaptchaScript() {
  if (!onRecaptchaScriptLoad) {
    const recaptchaScript = document.createElement('script');
    // TODO: modify the language of reCAPTCHA when we enable i18n
    const lang = 'en';
    recaptchaScript.src = 'https://www.google.com/recaptcha/api.js' +
      `?onload=recaptchaLoaded&render=explicit&hl=${lang}`;
    recaptchaScript.async = true;
    recaptchaScript.defer = true;

    onRecaptchaScriptLoad = new Promise((resolve) => {
      if (typeof window !== 'undefined') {
        // window.recaptchaLoaded will be called when reCaptcha script is
        // loaded
        window.recaptchaLoaded = resolve;
      }
    });
    document.head.appendChild(recaptchaScript);
  }
  return onRecaptchaScriptLoad;
}

export default {
  name: 'TimeSlots',
  components: {
    InfiniteLoading,
    TextInput,
    Textarea,
    InputLabel,
    Title,
    Button,
    TimeSlotBlock,
  },
  data() {
    return {
      step: 0,
      timeSlotsRenderOffset: 0,
      hasMoreTimeSlotsPages: true,
      stepTitles: [
        'Choose an available time slot',
        'What\'s your name and email?',
        'Confirm your meeting details',
      ],
      isTargetFormValid: true,
      timeZone: moment.tz.guess(),
      enableRecaptcha: false,
      recaptchaId: null,
    };
  },
  computed: mapState({
    meetingWindow: state => state.meetingWindow,
    timeSlots: state => state.timeSlots,
    loading: state => state.api.loading.timeSlots,
    launchOptions: state => state.launchOptions.schedule,
    slotGroups(state) {
      let availableSlots = state.timeSlots.availableSlots || [];
      availableSlots = availableSlots.slice(0, this.timeSlotsRenderOffset);

      const slotGroups = {};

      availableSlots.forEach((slot) => {
        const offsetStartDate = moment(slot.start);

        const groupKey = offsetStartDate.format('YYYY-MM-DD');
        if (typeof slotGroups[groupKey] === 'undefined') {
          slotGroups[groupKey] = [];
        }
        slotGroups[groupKey].push(slot);
      });

      return slotGroups;
    },
    visible: state => state.timeSlots.visible,
    isExtraDescriptionVisible: (state) => {
      const { visible } = state.timeSlots;
      const { allowEventMetadata } = state.meetingWindow;
      return allowEventMetadata && visible.extraDescription;
    },
  }),
  beforeMount() {
    if (!this.meetingWindow.id) {
      this.$store.dispatch({
        type: 'meetingWindow/getMeetingWindow',
        meetingWindowId: this.launchOptions.meetingWindowId,
      }).then(() => {
        if (this.meetingWindow.recaptchaSiteKey) {
          this.enableRecaptcha = true;
          loadRecaptchaScript().then(() => {
            const lang = 'en';
            this.recaptchaId = grecaptcha.render('recaptcha', {
              badge: 'bottomleft',
              sitekey: this.meetingWindow.recaptchaSiteKey,
              size: 'invisible',
              callback: this.onRecaptchaVerify,
              'error-callback': this.onRecaptchaError,
              isolated: true,
              lang,
            });
          });
        }
      });
    }
  },
  props: [
  ],
  methods: {
    selectTimeSlot(slot, selected) {
      this.$store.commit({
        type: 'timeSlots/selectTimeSlot',
        slot,
        selected,
      });
    },
    formatDate(format, dateStr) {
      return date(format, dateStr, this.timeZone);
    },
    moveStep(step) {
      this.step += step;
      this.timeSlotsRenderOffset = 0;
    },
    updateInput(event) {
      this.$store.commit({
        type: 'timeSlots/update',
        name: event.name,
        value: event.value,
      });
    },
    afterSubmit() {
      const { afterSchedule } = this.launchOptions;
      if (afterSchedule.showResult) {
        this.$router.push('/timeSlotsCompletion/');
      } else {
        this.$store.dispatch('event', {
          event: EVENTS.CLOSE,
        });
      }
    },
    submit(recaptchaToken) {
      const promise = this.$store.dispatch('timeSlots/submit', {
        recaptchaToken,
      });
      promise.then(() => this.afterSubmit());
    },
    validateForm() {
      if (this.$refs.form.validate()) {
        this.moveStep(1);
      }
    },
    /**
     * Run after reCAPTCHA successfully verified.
     *
     * @param {string} recaptchaToken - token returned by reCAPTCHA
     */
    onRecaptchaVerify(recaptchaToken) {
      this.submit(recaptchaToken);
    },
    /**
     * Run when any error is encountered by reCAPTCHA
     */
    onRecaptchaError() {
      // There is no error message from recaptcha, the common errors may be
      // invalid site key or no internet connection.
      const message = 'Error: reCAPTCHA error. Please try again or contact ' +
        'support.';
      this.$store.commit({
        type: 'api/setErrorMessage',
        message,
      });
    },
    executeRecaptcha() {
      if (this.enableRecaptcha) {
        onRecaptchaScriptLoad.then(
          () => grecaptcha.execute(this.recaptchaId),
        );
      } else {
        this.submit();
      }
    },
    /**
     * Render the next MAX_TIME_SLOTS_PER_SCROLL slots from the time slots list
     * in vuex state when user scrolls to the bottom.
     */
    async infiniteHandler(state) {
      let availableSlotsLength = this.timeSlots.availableSlots.length;
      if (
        (availableSlotsLength - this.timeSlotsRenderOffset
          < MAX_TIME_SLOTS_PER_SCROLL) && this.hasMoreTimeSlotsPages) {
        // if remain slots are not enough for this scroll and there is
        // a next page id from previous response, pull the next page.
        // This condition is also triggered when the view is loaded.
        this.hasMoreTimeSlotsPages = await this.$store.dispatch({
          type: 'timeSlots/getTimeSlots',
        });
      }

      // new time slots might be added, get the latest slot list length
      availableSlotsLength = this.timeSlots.availableSlots.length;

      if (availableSlotsLength) {
        // move render offset, view will be updated automatically
        // because the computed value `slotGroup` will be updated
        this.timeSlotsRenderOffset += MAX_TIME_SLOTS_PER_SCROLL;
        if (this.timeSlotsRenderOffset >= availableSlotsLength) {
          this.timeSlotsRenderOffset = availableSlotsLength;
        }
      }

      if (!availableSlotsLength && !this.hasMoreTimeSlotsPages) {
        // to trigger "no-results"
        state.complete();
      } else {
        state.loaded();
        if (this.timeSlotsRenderOffset === availableSlotsLength
            && !this.hasMoreTimeSlotsPages) {
          // disable the infinite scroll to fetch more time slots
          // and show the message about "no more items"
          state.complete();
        }
      }
    },
  },
};

/* eslint-disable */
</script>

<template lang="pug">
v-layout(column).time-slots
  div(v-if="loading.meetingWindow")
    Title Retrieving Event Details...
  div(v-else)
    Title {{ stepTitles[step] }}

  template(v-if="step === 0")
    div.timeslots-scroll-panel
      template(v-if="meetingWindow.id")
        div(v-for="(slots, date) in slotGroups", :key="date")
          div.text-xs-left.font-size--subtitle.secondary--text.font-weight-medium
            | {{ formatDate('date', date) }}
          v-layout(row, wrap, pa-3)
            .time-slot-block(v-for="(slot, index) in slots", :key="index")
              TimeSlotBlock(:timeSlot="slot", :timeZone="timeZone",
                            :formatDate="formatDate")
        InfiniteLoading(@infinite="infiniteHandler")
          div(slot="spinner")
            div.progress-container
              v-progress-circular(size="70", color="primary", indeterminate)
          div(slot="no-more").font-size--subtitle.secondary--text
            | No more time slots.
          div(slot='no-results').font-size--subtitle.secondary--text
            | Sorry. There are no available times for this event.
    div.pt-3.pb-4
      v-layout(row, justify-space-between)
        div.text-xs-left.subheading.surface--text
          | All times are in {{timeZone}} time
        Button(@click="moveStep(1)", :disabled="!timeSlots.selectedSlot").ma-0
          | Next

  template(v-if="step === 1")
    div.timeslots-scroll-panel
      v-form(ref="form", v-model="isTargetFormValid", lazy-validation)
        TextInput(
          v-if="visible.name",
          name="name", label="Name *", :value="timeSlots.name",
          required,
          placeholder="Johnny Appleseed", @update="updateInput")
        TextInput(
          v-if="visible.email",
          name="email", label="Email *", :value="timeSlots.email",
          required, type="email",
          placeholder="youremail@example.com", @update="updateInput")
        Textarea(v-if="isExtraDescriptionVisible"
          name="extraDescription" label="Note",
          :value="timeSlots.extraDescription",
          placeholder="Additional Information", @update="updateInput")
    div.pt-3.pb-4
      v-layout(row, wrap, justify-space-between)
        Button(@click="moveStep(-1)").ma-0
          | Back
        Button(@click="validateForm", :disabled="!isTargetFormValid").ma-0
          | Next

  template(v-if="step === 2")
    div.timeslots-scroll-panel.text-xs-left
      div.font-size--subtitle.secondary--text.font-weight-bold {{ meetingWindow.title }}
      div.mb-5.on-primary--text.font-size--md {{ meetingWindow.location }}
      InputLabel.mb-1 SCHEDULED TIME
      div.mb-4
        div.font-size--lg.secondary--text.font-weight-bold
          | {{ formatDate('fullHour', timeSlots.selectedSlot.start) }}
          | -
          | {{ formatDate('fullHour', timeSlots.selectedSlot.end) }}
          | ({{timeZone}} time)
        div.font-size--md.secondary--text
          | {{ formatDate('date', timeSlots.selectedSlot.start) }}
      InputLabel.mb-1 CONTACT INFO
      div.mb-4
        div.font-size--lg.secondary--text.font-weight-bold {{ timeSlots.name }}
        div.font-size--md.secondary--text {{ timeSlots.email }}
      Textarea(v-if="isExtraDescriptionVisible && timeSlots.extraDescription"
               name="extraDescription" label="Note",
               :value="timeSlots.extraDescription", :readonly="true")
    div.pt-3.pb-4
      v-layout(row, wrap, justify-space-between)
        Button(@click="moveStep(-1)").ma-0 Back
        Button(@click="executeRecaptcha" :loading="loading.submit").ma-0
          | Schedule Meeting
  div#recaptcha
</template>
