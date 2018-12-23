<template>
  <div class="vue-phone-number" :class="{ disabled: disabled }">
    <CountrySelect :items="sortedCountries" v-model="countryCode"/>
    <input
      ref="input"
      type="tel"
      v-bind="$attrs"
      v-on="$listeners"
      v-model="phone"
      :placeholder="placeholder"
      :state="state"
      :disabled="disabled"
      :required="required"
      @input="onInput"
    >
  </div>
</template>

<script>
import { formatNumber, AsYouType, isValidNumber } from 'libphonenumber-js'
import countryTelData from 'country-telephone-data'
import { getCountry } from './utils'
import CountrySelect from './CountrySelect.vue'

const allCountries = countryTelData.allCountries
const iso2Lookup = countryTelData.iso2Lookup
let userCountryCode

export default {
  name: 'VuePhoneNumber',
  components: {
    CountrySelect
  },
  props: {
    value: String,
    placeholder: {
      type: String,
      default: () => 'Enter a phone number'
    },
    disabledFetchingCountry: {
      type: Boolean,
      default: () => false
    },
    disabled: {
      type: Boolean,
      default: () => false
    },
    invalidMsg: {
      default: () => '',
      type: String
    },
    required: {
      type: Boolean,
      default: () => false
    },
    defaultCountry: {
      // Default country code, ie: 'AU'
      // Will override the current country of user
      type: String,
      default: ''
    },
    enabledFlags: {
      type: Boolean,
      default: true
    },
    preferredCountries: {
      type: Array,
      default: () => []
    },
    onlyCountries: {
      type: Array,
      default: () => []
    },
    ignoredCountries: {
      type: Array,
      default: () => []
    }
  },
  mounted () {
    this.initializeCountry()
  },
  created () {
    if (this.value) {
      this.phone = this.value
    }
  },
  data () {
    return {
      phone: '',
      countryCode: null
    }
  },
  computed: {
    mode () {
      if (!this.phone) {
        return ''
      }
      if (this.phone[0] === '+') {
        return 'code'
      }
      if (this.phone[0] === '0') {
        return 'prefix'
      }
      return 'normal'
    },
    filteredCountries () {
      // List countries after filtered
      if (this.onlyCountries.length) {
        return this.getCountries(this.onlyCountries)
      }

      if (this.ignoredCountries.length) {
        return allCountries.filter(({ iso2 }) => !this.ignoredCountries.includes(iso2))
      }

      return allCountries
    },
    sortedCountries () {
      // Sort the list countries: from preferred countries to all countries
      const preferredCountries = this.getCountries(this.preferredCountries)
        .map(country => ({ ...country, preferred: true }))

      return [
        ...preferredCountries,
        ...this.filteredCountries
      ]
    },
    formattedResult () {
      // Calculate phone number based on mode
      if (!this.mode || !this.filteredCountries) {
        return ''
      }
      let phone = this.phone
      if (this.mode === 'code') {
        // If user manually type the country code
        const formatter = new AsYouType()// eslint-disable-line
        formatter.input(this.phone)
        const countryCode = formatter.country && formatter.country.toLowerCase()
        // console.log(formatter)
        // Find inputted country in the countries list
        if (countryCode) {
          this.countryCode = countryCode // eslint-disable-line
        }
      } else if (this.mode === 'prefix') {
        // Remove the first '0' if this is a '0' prefix number
        // Ex: 0432421999
        phone = this.phone.slice(1)
      }

      return formatNumber(phone, this.counrtyCodeUC, 'INTERNATIONAL')
    },
    state () {
      return isValidNumber(this.formattedResult, this.counrtyCodeUC)
    },
    response () {
      // If it is a valid number, returns the formatted value
      // Otherwise returns what it is
      const number = this.state ? this.formattedResult : this.phone
      return {
        number,
        isValid: this.state,
        country: allCountries[iso2Lookup[this.countryCode]]
      }
    },
    counrtyCodeUC () {
      return this.countryCode && this.countryCode.toUpperCase()
    }
  },
  watch: {
    state (value) {
      if (value && this.mode !== 'prefix') {
        // If mode is 'prefix', keep the number as user typed,
        // Otherwise format it
        this.phone = this.formattedResult
      }
    },
    value () {
      this.phone = this.value
    }
  },
  methods: {
    initializeCountry () {
      /**
       * 1. Use default country if passed from parent
       */
      if (this.defaultCountry) {
        this.countryCode = this.defaultCountry.toLowerCase()
      }
      /**
       * 2. Use the first country from preferred list (if available) or all countries list
       */
      this.countryCode = this.preferredCountries[0] || this.filteredCountries[0].iso2
      /**
       * 3. Check if fetching country based on user's IP is allowed, set it as the default country
       */
      if (!this.disabledFetchingCountry) {
        if (!userCountryCode) {
          getCountry().then((res) => {
            userCountryCode = res.toLowerCase()
            this.countryCode = userCountryCode
            // this.preferredCountries.push(userCountryCode)
          })
        } else {
          this.countryCode = userCountryCode
          // this.preferredCountries.push(userCountryCode)
        }
      }
    },
    /**
     * Get the list of countries from the list of iso2 code
     */
    getCountries (list = []) {
      return list
        .map(countryCode => allCountries[iso2Lookup[countryCode]])
        .filter(Boolean)
    },
    onInput (value) {
      this.$refs.input.setCustomValidity(this.response.isValid ? '' : this.invalidMsg)
      // Emit input event in case v-model is used in the parent
      this.$emit('input', this.response.number)
      // Emit the response, includes phone, validity and country
      this.$emit('change', this.response)
    }
  }
}
</script>

<style scoped>
:local {
  --border-radius: 2px
}

.vue-phone-number {
  /* border-radius: 3px */
  display: flex;
  /* border: 1px solid #bbb */
  text-align: left;
}
.vue-phone-number:focus-within {
  box-shadow: inset 0 1px 1px rgba(0, 0, 0, 0.075),
    0 0 8px rgba(102, 175, 233, 0.6);
  border-color: #66afe9;
}
input {
  border: none;
  border-radius: 0 var(--border-radius) var(--border-radius) 0;
  width: 100%;
  outline: none;
  padding-left: 7px;
}

.vue-phone-number.disabled .selection,
.vue-phone-number.disabled .dropdown,
.vue-phone-number.disabled input {
  cursor: no-drop;
}
</style>
