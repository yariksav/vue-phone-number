<template>
  <div
    class="dropdown"
    @click="toggleDropdown"
    v-click-outside="clickedOutside"
    :class="{open: open}"
    tabindex="0"
    @keydown.exact="keyboardNav"
    @keydown.esc="reset"
  >
    <span class="selection">
        <VueFlag :code="selected" size="small"/>
      <span class="dropdown-arrow">{{ open ? '▲' : '▼' }}</span>
    </span>
    <ul v-show="open" ref="list">
      <li
        class="dropdown-item"
        v-for="(item, index) in items"
        :key="item.iso2 + (item.preferred ? '-preferred' : '')"
        @click="onChange(item)"
        :class="getItemClass(index, item.iso2)"
        @mousemove="selectedIndex = index"
      >
        <VueFlag :code="item.iso2" size="small" />
        <strong>{{ item.name }}</strong>
        <span>+{{ item.dialCode }}</span>
      </li>
    </ul>
  </div>
</template>
<script>
import VueFlag from 'vue-flag'
import 'vue-flag/dist/vue-flag.css'

export default {
  components: {
    VueFlag
  },
  props: {
    items: Array,
    value: String,
    preferredCountries: {
      type: Array,
      default: () => []
    }
  },
  data () {
    return {
      open: false,
      selected: this.value,
      selectedIndex: -1
    }
  },
  watch: {
    value (value) {
      this.selected = value
    }
  },
  methods: {
    onChange (country) {
      this.selected = country.iso2
      this.$emit('input', country.iso2)
    },
    getItemClass (index, iso2) {
      const highlighted = this.selectedIndex === index
      const lastPreferred = index === this.preferredCountries.length - 1
      const preferred = !!~this.preferredCountries.map(c => c.toLowerCase()).indexOf(iso2)
      return {
        highlighted,
        'last-preferred': lastPreferred,
        preferred
      }
    },
    toggleDropdown () {
      if (this.disabled) {
        return
      }
      this.open = !this.open
    },
    clickedOutside () {
      this.open = false
    },
    keyboardNav (e) {
      // const list = this.$refs.list
      if (e.keyCode === 40) {
        // down arrow
        this.open = true
        if (this.selectedIndex === null) {
          this.selectedIndex = 0
        } else {
          this.selectedIndex = Math.min(this.items.length - 1, this.selectedIndex + 1)
        }
        let selEle = this.$refs.list.children[this.selectedIndex]
        if (selEle.offsetTop + selEle.clientHeight > this.$refs.list.scrollTop + this.$refs.list.clientHeight) {
          this.$refs.list.scrollTop = selEle.offsetTop - this.$refs.list.clientHeight + selEle.clientHeight
        }
      } else if (e.keyCode === 38) {
        // up arrow
        this.open = true
        if (this.selectedIndex === null) {
          this.selectedIndex = this.items.length - 1
        } else {
          this.selectedIndex = Math.max(0, this.selectedIndex - 1)
        }
        let selEle = this.$refs.list.children[this.selectedIndex]
        if (selEle.offsetTop < this.$refs.list.scrollTop) {
          this.$refs.list.scrollTop = selEle.offsetTop
        }
      } else if (e.keyCode === 13) {
        // enter key
        if (this.selectedIndex !== null) {
          this.onChange(this.items[this.selectedIndex])
        }
        this.open = !this.open
      } else {
        // typing a country's name
        this.typeToFindInput += e.key
        clearTimeout(this.typeToFindTimer)
        this.typeToFindTimer = setTimeout(() => {
          this.typeToFindInput = ''
        }, 700)
        // don't include preferred countries so we jump to the right place in the alphabet
        let typedCountryI = this.items.slice(this.preferredCountries.length).findIndex(c => c.name.toLowerCase().startsWith(this.typeToFindInput))
        if (~typedCountryI) {
          this.selectedIndex = this.preferredCountries.length + typedCountryI
          let selEle = this.$refs.list.children[this.selectedIndex]
          if (selEle.offsetTop < this.$refs.list.scrollTop || selEle.offsetTop + selEle.clientHeight > this.$refs.list.scrollTop + this.$refs.list.clientHeight) {
            this.$refs.list.scrollTop = selEle.offsetTop - this.$refs.list.clientHeight / 2
          }
        }
      }
    },
    reset () {
      this.selectedIndex = this.items.map(c => c.iso2).indexOf(this.selected)
      this.open = false
    }
  },
  directives: {
    // Click-outside from BosNaufal: https://github.com/BosNaufal/vue-click-outside
    'click-outside': {
      bind: function (el, binding, vNode) {
        // Provided expression must evaluate to a function.
        if (typeof binding.value !== 'function') {
          var compName = vNode.context.name
          var warn = '[Vue-click-outside:] provided expression ' + binding.expression + ' is not a function, but has to be'
          if (compName) {
            warn += 'Found in component ' + compName
          }
          console.warn(warn); /* eslint-disable-line */
        }
        // Define Handler and cache it on the element
        var bubble = binding.modifiers.bubble
        var handler = function (e) {
          if (bubble || (!el.contains(e.target) && el !== e.target)) {
            binding.value(e)
          }
        }
        el.__vueClickOutside__ = handler
        // add Event Listeners
        document.addEventListener('click', handler)
      },
      unbind: function (el) {
        // Remove Event Listeners
        document.removeEventListener('click', el.__vueClickOutside__)
        el.__vueClickOutside__ = null
      }
    }
  }
}
</script>

<style scoped>

li.last-preferred {
  border-bottom: 1px solid #cacaca;
}
.selection {
  font-size: 0.8em;
  display: flex;
  align-items: center;
}

ul {
  z-index: 1;
  padding: 0;
  margin: 0;
  text-align: left;
  list-style: none;
  max-height: 200px;
  overflow-y: scroll;
  position: absolute;
  top: 33px;
  left: -1px;
  background-color: #fff;
  border: 1px solid #ccc;
  width: 390px;
}
.dropdown {
  display: flex;
  flex-direction: column;
  align-content: center;
  justify-content: center;
  position: relative;
  padding: 7px;
  cursor: pointer;
}
.dropdown:focus {
    outline: none
}
/* .dropdown.open {
  background-color: #f3f3f3;
}
.dropdown:hover {
  background-color: #f3f3f3;
} */
.dropdown-arrow {
  transform: scaleY(0.5);
  display: inline-block;
  color: #666;
}
.dropdown-item {
  cursor: pointer;
  padding: 4px 15px;
}
.dropdown-item.highlighted {
  background-color: #f3f3f3;
}
.dropdown-menu.show {
  max-height: 300px;
  overflow: scroll;
}
</style>
