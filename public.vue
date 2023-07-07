<script>
import { mapState } from 'vuex'

export default {
  name: 'ModalAddToCart',

  props: {
    product: {
      type: Object,
      required: true
    },
    related: {
      type: Object,
      default: null
    }
  },

  data () {
    return {
      quantity: 1,
      value: [],
      valueFinal: [],
      guaranteeValue: null,
      priceGuarantee: null,
      priceInsurances: null,
      loadingContinue: null,
      loading: false,
      limitExceeded: false
    }
  },

  computed: {
    ...mapState({
      insurances: state => state.product.insurances,
      guarantee: state => state.product.guarantee,
      services: state => state.product.services,
      cart: state => state.cart.cart
    }),

    defectiveTitle () {
      return this.product.isDefective ? ' (lorem)' : ''
    },

    title () {
      if (this.product.groupedModifications && this.product.groupedModifications.length) {
        const color = this.product.groupedModifications
          .find(mod => mod.title === 'Цвет')?.options
          .find(opt => opt.slug === this.product.slug)

        if (color) {
          return `${this.product.name} цвет ${color.title}` + this.defectiveTitle
        }
      }
      return this.product.name + this.defectiveTitle
    },

    options () {
      const insurances = this.product.insurances ? [...this.product.insurances].slice(3, 5) : null
      const services = this.product.services ? [...this.product.services].sort(() => Math.random() - 0.5).slice(0, 2) : null
      const result = [
        {
          name: 'lorem',
          description: 'lorem' +
            'lorem'
        },
        {
          name: 'lorem',
          description: 'lorem'
        }
      ]
      if (insurances) {
        result.push(...insurances)
      }
      if (services) {
        result.push(...services)
      }
      return result
    },

    guaranteesInCard () {
      return this.product.guarantees ? [...this.product.guarantees].slice(0, 3) : null
    },

    insurancesInCard () {
      return this.product.insurances ? [...this.product.insurances].slice(0, 3) : null
    },

    otherOptions () {
      return [...this.options].slice(2)
    },

    productId () {
      const arr = []
      for (const item in this.cart) {
        if (this.cart[item].checked) {
          arr.push(this.cart[item].id)
        }
      }
      return arr
    }
  },

  watch: {
    quantity () {
      if (this.quantity > this.product.quantityAvailable) {
        this.quantity--
        this.limitExceeded = true
      }
    }
  },

  mounted () {
    this.guaranteeValue = this.guarantee ? this.guarantee.uuid : null
    this.priceGuarantee = this.guarantee ? this.guarantee.price : null
    if (this.guaranteeValue) {
      this.value.push(this.options[0])
      this.valueFinal.push(this.guarantee)
    }

    if (this.insurances.length) {
      this.value.push(...this.insurances)
      this.valueFinal.push(...this.insurances)
      this.value.push(this.options[1])
      this.priceInsurances = this.insurances.slice(0, 3).reduce((acc, value) => acc + Number(value.price), 0)
    }

    if (this.services.length) {
      this.value.push(...this.services)
      this.valueFinal.push(...this.services)
    }
  },

  methods: {
    increment () {
      this.quantity += 1
    },

    decrement () {
      this.quantity = this.quantity > 1 ? this.quantity -= 1 : 1
    },

    checked (name) {
      return Boolean(this.value.find(item => item.name === name))
    },

    change (selected, checked) {
      if (checked) {
        selected.checked = 1
        this.value.push(selected)
        if (selected.name === 'lorem') {
          this.guaranteeValue = this.guaranteesInCard[0].uuid
          this.priceGuarantee = this.guaranteesInCard[0].price
        } else if (selected.name === 'lorem') {
          this.insurancesInCard[0].checked = 1
          this.value.push(this.insurancesInCard[0])
          this.priceInsurances += Number(this.insurancesInCard[0].price)
        }
      } else {
        selected.checked = 0
        if (selected.name === 'lorem') {
          this.guaranteeValue = null
          this.priceGuarantee = null
        } else if (selected.name === 'lorem') {
          this.value = this.value.filter(item => !item.name.includes('lorem'))
          this.insurancesInCard.forEach(el => (el.checked = 0))
          this.priceInsurances = null
        }
        this.value = this.value.filter(item => item.name !== selected.name)
      }
    },

    checkedCombo (uuid) {
      return Boolean(this.value.find(item => item.uuid === uuid))
    },

    changeCombo (selected, checked) {
      if (checked) {
        selected.checked = 1
        this.value.push(selected)
        this.value = this.value.filter(item => (item.name !== 'lorem'))
        this.value.push(this.options[1])
        this.priceInsurances += Number(selected.price)
      } else {
        selected.checked = 0
        this.value = this.value.filter(item => item.checked !== selected.checked)
        this.priceInsurances -= Number(selected.price)
        if (this.priceInsurances === 0) {
          this.value = this.value.filter(item => (item.name !== 'lorem'))
        }
      }
    },

    changeValue (selected, name) {
      if (selected && name === 'lorem') {
        this.priceGuarantee = selected.price
        this.value = this.value.filter(item => item.name !== this.options[0].name)
        this.value.push(this.options[0])
      }
    },

    cost (cost) {
      return cost === '0' ? 'бесплатно' : this.$options.filters.price(cost)
    },

    add (toggle) {
      if (toggle === 'cart') {
        this.loading = true
      } else {
        this.loadingContinue = true
      }

      this.$emit('add')
      this.value = this.value.filter(item => (item.name !== 'lorem' && item.name !== 'lorem'))

      const obj = {}

      for (const key of [...this.product.guarantees]) {
        if (key.uuid === this.guaranteeValue) {
          obj[key.uuid] = this.quantity
          this.$store.commit('product/setGuarantee', key)
        } else {
          obj[key.uuid] = 0
        }
      }

      if (Object.values(obj).find(item => item === 0)) {
        this.$store.commit('product/setGuarantee', null)
      }

      const insurance = this.value.filter(item => [...this.product.insurances].some(el => item.name === el.name))
      this.$store.commit('product/setInsurances', insurance)

      const service = this.value.filter(item => [...this.product.services].some(el => item.name === el.name))
      this.$store.commit('product/setServices', service)

      this.$store.dispatch('cart/removeFromCart', this.valueFinal)

      const optionsAdd = [...this.value]

      this.$store.dispatch('cart/addProduct', {
        uuid: this.product.uuid,
        id: this.productId,
        qt: this.quantity,
        i: { ...this.productId },
        optionsAdd,
        guarantee: obj
      })
        .finally(() => {
          if (toggle === 'cart') {
            this.loading = false
            window.location.href = 'https://lorem.ru'
          } else {
            this.loadingContinue = false
            this.$root.$emit('hide-modal')
          }
        })
    }
  }
}
</script>

<template>
  <ModalBase
    size="xxxxl"
  >
    <div
      :class="$style.header"
    >
      <h1 :class="$style.heading">
        lorem
      </h1>
      <div>
        <!-- Continue Button -->
        <BaseBtn
          :class="$style.btn"
          mode="outline"
          color="primary"
          :loading="loadingContinue"
          @click="add('hide')"
        >
          Продолжить покупки
        </BaseBtn>

        <!-- Cart Button -->
        <BaseBtn
          :class="$style.btn"
          mode="fill"
          color="primary"
          :loading="loading"
          @click="add('cart')"
        >
          Перейти в корзину
        </BaseBtn>
      </div>
    </div>

    <!-- Product -->
    <div
      :class="$style.product"
    >
      <div :class="$style.item">
        <img
          :class="$style.img"
          :src="$img(product.images[0].template.replace('PHOTO_SIZE', '150x150'))"
          alt=""
        >
        <div :class="$style.description">
          <h2 :class="$style.title">
            {{ title }}
          </h2>
          <div :class="$style.deliveries">
            <div :class="$style.availability">
              {{ product.available ? 'В наличии' : 'Не в наличии' }}
            </div>
            <BaseBtn
              v-if="product.availableShops.length"
              :class="$style.shops"
              color="link"
              @click="$root.$emit('show-modal', 'availability', { uuid: product.uuid })"
            >
              из {{ product.availableShops.length }} {{ 'магазин' | declension(product.availableShops.length, 'а', 'ов', 'ов') }}
            </BaseBtn>
            <div v-else>
              {{ product.allDeliveries[0].title }} {{ product.allDeliveries[0].deliveryPeriod.slice(8) }}, {{ cost(product.allDeliveries[0].deliveryCost) }}
            </div>
            <div v-if="product.allDeliveries[1]">
              {{ product.allDeliveries[1].title }} {{ product.allDeliveries[1].deliveryPeriod }}, {{ cost(product.allDeliveries[1].deliveryCost) }}
            </div>
          </div>
        </div>
      </div>

      <!-- Price -->
      <div :class="$style.price">
        <div :class="$style.quantity">
          <div :class="$style.counter">
            <BaseBtn
              @click="decrement"
            >
              -
            </BaseBtn>
            <span>{{ quantity }}</span>
            <BaseBtn
              @click="increment"
            >
              +
            </BaseBtn>
          </div>
          <div
            v-if="limitExceeded"
            :class="$style.limit"
          >
            В наличии {{ product.quantityAvailable }} шт.
          </div>
        </div>
        <div :class="$style.sum">
          <span>Цена</span> {{ quantity * product.price | price }}
        </div>
      </div>
    </div>

    <!-- Services -->
    <template v-if="product.insurances || product.services || product.guarantees">
      <div :class="$style.line" />
      <div :class="$style.useful">
        <h3 :class="$style.subheading">
          Полезные услуги
        </h3>
        <div :class="$style.services">
          <div :class="$style.list">
            <div
              v-for="(service, index) in options"
              :key="index"
              :class="$style.name"
            >
              <BaseCheckbox
                :checked="checked(service.name)"
                @change="change(service, $event)"
              >
                {{ service.name }}
              </BaseCheckbox>
              <div
                v-tooltip="{
                  content: service.description,
                  classes: 'tooltip-modal',
                }"
              >
                <SvgIcon
                  name="20/question-gray.svg"
                />
              </div>
            </div>
          </div>
          <div
            :class="$style.options"
          >
            <div :class="$style.row">
              <BaseRadio
                v-for="guarantee in guaranteesInCard"
                :key="guarantee.name"
                v-model="guaranteeValue"
                :value="guarantee.uuid"
                @change="changeValue(guarantee, 'guarantee')"
              >
                {{ guarantee.name.slice(0, 8) }}
              </BaseRadio>
              <span>{{ quantity * priceGuarantee | price }}</span>
            </div>
            <div :class="$style.row">
              <BaseCheckbox
                v-for="insurances in insurancesInCard"
                :key="insurances.uuid"
                :checked="checkedCombo(insurances.uuid)"
                @change="changeCombo(insurances, $event)"
              >
                {{ insurances.name.slice(5, 15) }}
              </BaseCheckbox>
              <span>{{ quantity * priceInsurances | price }}</span>
            </div>
            <div
              v-for="option in otherOptions"
              :key="option.name"
              :class="$style.row"
            >
              <span>{{ quantity * option.price | price }}</span>
            </div>
          </div>
        </div>
      </div>
      <div :class="$style.line" />
    </template>

    <!-- Related Block -->
    <ProductCompilation
      v-if="related"
      :class="$style.related"
      :title="related.title"
      :category-id="related.slug"
      :product-articul="product.articul"
    />
  </ModalBase>
</template>

<style lang="scss" module>

.header {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 36px;
}

.heading {
  margin: 0;
  @include font(heading-1, extrabold)
}

.btn {
  width: 230px;

  &:nth-child(1) {
    margin-right: 24px;
  }
}

.product {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  margin-bottom: 32px;
}

.line {
  border: 1px solid var(--kc-color-gray-400);
  margin: 0 -48px;
}

.item {
  display: flex;
  max-width: 500px;
}

.img {
  margin-right: 26px;
}

.title {
  margin-bottom: 8px;
  @include font(heading-2, extrabold);
  line-height: 32px;
}

.deliveries {
  display: flex;
  flex-wrap: wrap;
}

.availability {
  margin-right: 5px;
}

.price {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.quantity {
  display: flex;
  flex-direction: column;
  align-items: center;
  height: 50px;
  width: 100px;
  margin-top: 23px;
  margin-right: 20px;
}

.counter {
  display: inline-flex;
  justify-content: space-between;
  align-items: center;
  width: 92px;
  height: 32px;
  background-color: var(--kc-color-bg-light);
  border-radius: 4px;
  padding: 0 8px;
}

.limit {
  font-size: var(--kc-font-size-caption-1);
  line-height: var(--kc-line-height-caption-1);
  color: var(--kc-color-gray-500);
  text-align: center;
}

.sum {
  width: 300px;
  text-align: right;
  @include font(heading-1, extrabold);

  &>span {
    margin-right: 24px;
    @include font(heading-2, extrabold)
  }
}

.useful {
  margin: 24px 0;
}

.services {
  display: flex;
  justify-content: space-between;
}

.subheading {
  @include font(subheading-1);
  margin-bottom: 24px;
}

.list {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.name {
  display: flex;
  flex-direction: row;
  gap: 8px;
}

.options {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.row {
  display: flex;
  flex-direction: row;
  justify-content: end;
  gap: 12px;

  &>span {
    display: block;
    width: 70px;
    margin-left: 64px;
    text-align: right;
    @include font(body-2, bold)
  }
}

.related {
  max-width: 1055px;
  margin-top: 32px;
}

</style>

<style lang="scss">

.tooltip-modal {
  z-index: 100;

  .tooltip-inner {
    @include font(caption-1);
    padding: 12px 18px;
    border-radius: 8px;
    color: var(--kc-color-text);
    background-color: var(--kc-color-bg-light);
    box-shadow: 0 0 12px 0 rgba(34, 60, 80, 0.46);
  }

  .tooltip-arrow {
    border-top-color: var(--kc-color-bg-light)!important;
  }

}

</style>
