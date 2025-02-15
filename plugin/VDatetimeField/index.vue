<template lang="pug">
  v-input.v-datetime-field(
    v-bind="commonAttrs"
    :error-count="commonAttrs['error-messages'].length"
    :value="outputValue"
  )
    .v-datetime-field__wrapper
      .v-datetime-field__date(v-if="!onlyTime")
        v-menu(
          v-model="date.menu"
          v-bind="mixedMenuProps"
        )
          template(v-slot:activator="{ on }")
            v-text-field(
              v-model="date.textField"
              append-icon="mdi-calendar"
              v-bind="dateProps"
              v-on="on"
              type="text"
              v-mask="'##.##.####'"
              placeholder="__.__.____"
              hide-details
              @click:append="openDate"
              @click:clear="date.textField = null"
              @keyup.enter="timeFocus"
              @keydown.tab="timeFocus"
              @click="openDate"
              @blur="emitValue"
            )
          v-date-picker(
            v-model="date.picker"
            no-title
            scrollable
            @input="date.menu = false"
            @click:date="timeFocus"
          )
            v-spacer
            v-btn(text color="primary" @click="date.menu = false") Abbrechen

      .v-datetime-field__time(v-if="!onlyDate")
        v-menu(
          v-model="time.menu"
          v-bind="mixedMenuProps"
        )
          template(v-slot:activator="{ on }")
            v-text-field(
              ref="timePickerInput"
              v-model="time.textField"
              v-bind="timeProps"
              v-on="on"
              append-icon="mdi-clock"
              type="text"
              v-mask="'##:##'"
              placeholder="__:__"
              :class="{ 'ml-2': !onlyTime }"
              hide-details
              @click:append="openTime"
              @click:clear="time.textField = null"
              @keyup.enter="time.menu = false"
              @click="openTime"
              @blur="emitValue"
            )
          v-time-picker(
            :value="time.picker.value"
            format="24hr"
            @change="setTimePickerValue"
            @click:hour="setTimePickerValue"
          )
            v-spacer
            v-btn(text color="primary" @click="time.menu = false") Abbrechen

    template(
      v-for="(_, name) in $scopedSlots"
      v-slot:[name]="data"
    )
      slot(:name="name" v-bind="data")
</template>

<script>
import { parseISO, format, parse } from 'date-fns';
import { mask } from 'vue-the-mask';

const DEFAULT_FORMAT_DATE = 'dd.MM.yyyy';

const DEFAULT_MENU_PROPS = {
  'min-width': 290,
  'offset-y': true,
  'close-on-content-click': false,
  transition: 'scale-transition',
};

export default {
  name: 'DatePicker',
  directives: {
    mask,
  },
  props: {
    value: { type: String, default: null },

    onlyDate: { type: Boolean, default: false },
    onlyTime: { type: Boolean, default: false },

    dateProps: { type: Object, default: () => ({}) },
    timeProps: { type: Object, default: () => ({}) },
    menuProps: { type: Object, default: () => ({}) },
  },
  data() {
    return {
      date: {
        menu: false,
        textField: null,
        picker: null,
        validate: {
          rule: (v) =>
            /(0[1-9]|[12]\d|3[01]).(0[1-9]|1[0-2]).([12][0-9]{3})/.test(v),
          success: true,
        },
      },

      time: {
        menu: false,
        textField: null,
        picker: {
          value: null,
          fullfilled: false,
        },
        validate: {
          rule: (v) => /([0-1]\d|2[0-3]):[0-5][0-9]/.test(v),
          success: true,
        },
      },
    };
  },
  computed: {
    mixedMenuProps() {
      return { ...DEFAULT_MENU_PROPS, ...this.menuProps };
    },
    outputValue() {
      const [date, time] = [this.date.picker, this.time.picker.value];

      if (!this.onlyDate && !this.onlyTime) {
        return date && time ? [date, time].join(' ').trim() : '';
      }
      return [date, time].join(' ').trim() || null;
    },
    commonAttrs() {
      const { $attrs } = this || {};
      const localDatetimeErrors = [];
      if (!this.date.validate.success)
        localDatetimeErrors.push('Datum falsch eingegeben');
      if (!this.time.validate.success)
        localDatetimeErrors.push('Время введено неверно');
      return {
        ...$attrs,
        'error-messages': [
          ...($attrs['error-messages'] || []),
          ...localDatetimeErrors,
        ],
      };
    },
  },
  watch: {
    value: {
      handler(val) {
        const datetime = (val && val.split(' ')) || [val, val];
        let [date, time] = datetime;

        if (datetime.length === 1) {
          const [firstValue] = datetime;
          date = !firstValue.includes(':') ? firstValue : null;
          time = firstValue.includes(':') ? firstValue : null;
        }

        if (date === null) this.date.picker = date;
        if (time === null) this.time.picker = { value: time, fullfilled: !val };
      },
      immediate: true,
    },
    outputValue: function (val) {
      this.emitValue();
    },
    'date.picker': {
      handler(val) {
        let date = null;
        if (val) {
          date = parseISO(val);
          date = format(date, DEFAULT_FORMAT_DATE);
        }

        this.date.textField = date;
      },
      immediate: true,
    },
    'date.textField': function (val) {
      if ((val && val.length === 10) || !val) {
        this.date.validate.success = !val ? true : this.date.validate.rule(val);

        if (this.date.validate.success) {
          this.date.picker = val && this.setDate(val);
        } else {
          this.date.menu = false;
        }
      }
    },
    'time.picker.value': {
      handler(val) {
        this.time.textField = val;
      },
      immediate: true,
    },
    'time.textField': function (val) {
      let valFormatted = null;

      if (val && val.length === 5) {
        valFormatted = val;
      }

      if (valFormatted) {
        this.time.validate.success = this.time.validate.rule(valFormatted);

        if (this.time.validate.success) {
          this.time.picker = {
            value: valFormatted,
            fullfilled: val.length === 5,
          };
        } else {
          this.time.menu = false;
        }
      }
      if (!val) {
        this.time.picker = {
          value: null,
          fullfilled: true,
        };
      }
    },
  },
  methods: {
    setDate(val) {
      return format(parse(val, DEFAULT_FORMAT_DATE, new Date()), 'yyyy-MM-dd');
    },
    setTimePickerValue(val) {
      if (typeof val == 'number') {
        this.setHours(val);
      } else {
        this.time.picker = {
          value: val,
          fullfilled: true,
        };
        this.time.menu = false;
      }
    },
    setHours(val) {
      const hours = val > 9 ? val : `0${val}`;

      if (this.time.textField) {
        const [, minutes] = this.time.textField.split(':');
        this.time.textField = `${hours}:${minutes}`;
      } else {
        this.time.textField = `${hours}:00`;
      }
    },
    emitValue() {
      this.$emit('input', this.outputValue);
    },
    openDate() {
      if (this.date.validate.success) {
        this.date.menu = true;
        this.time.menu = false;
      }
    },
    openTime() {
      if (this.time.validate.success) {
        this.time.menu = true;
        this.date.menu = false;
      }
    },
    timeFocus() {
      const { timePickerInput } = this.$refs;
      this.date.menu = false;

      if (timePickerInput) {
        this.$nextTick(() => {
          setTimeout(() => {
            timePickerInput.focus();
          });
        });
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.v-datetime-field {
  &.v-input ::v-deep {
    & > .v-input__control {
      & > .v-input__slot {
        display: block !important;
      }

      & > .v-messages {
        padding: 0 12px;
        margin-bottom: 8px;
      }
    }
  }

  &__wrapper {
    display: flex;
    margin-top: 4px;
  }

  &__date {
    flex: 1;
  }

  &__time {
    ::v-deep .v-text-field input {
      max-width: 100px;
    }
  }
}
</style>
