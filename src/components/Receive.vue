<template lang="pug">
  div
    HCE(:accountNumber='total')
    template(v-if='generated')
      template(v-if='received')
        video(v-show='!finished' width='100%' ref='connect' @ended='finish')
          source(src="static/connect.mp4" type="video/mp4")
        v-alert(value='received' color='success') Received {{received}} Satoshis
      v-layout(v-else)
        v-flex(xs12)
          h2.text-xs-center Requesting {{total}} Satoshis
          v-card.pa-3.text-xs-center
            div.code(v-if='showcode') {{payreq}}
            canvas#qr(v-show='!showcode')
            v-btn(@click.native="showcode = !showcode")
              v-icon code
              span {{code}}
            v-btn(:data-clipboard-text='payreq' @click.native="snack('Copied to Clipboard')")
              v-icon content_copy
              span Copy
      v-btn(@click='generated = false; amount = 0') 
        v-icon.mr-1 arrow_back
        span Back
    template(v-else)
      v-layout(style="max-width: 380px")
        v-flex(xs9)
          numpad(:currency='currency' :amount='parseFloat(amount)' @update='a => amount = a')
        v-flex(xs3)
          tippad(:amount='parseFloat(amount)' @update='t => tip = t')
      div(v-if='valid')
        v-layout
          v-flex
            span.display-1 {{total}} 
            span.title sats
          v-flex.text-xs-right
            v-btn.darken-4.subheading.grey.white--text(@click='toggle' style='width: 120px') 
              v-avatar 
                span(v-html='symbol(currency)')
              span {{conversion}} 
        v-layout
          v-flex.text-xs-center(xs12)
            v-btn(@click='generate') 
              v-icon.mr-1.yellow--text mdi-flash
              span GO
      v-alert(v-else value='!valid') Can't request more than 4294967 satoshis
</template>

<script>
import qr from 'qrcode'
import numpad from './NumPad'
import tippad from './TipPad'
import HCE from './HCE'
import Lightning from './Lightning'
import { mapGetters, mapActions } from 'vuex'

const f = parseFloat

export default {
  components: { numpad, tippad, HCE, Lightning },

  filters: {
  },

  data () {
    return {
      message: '',
      about: '',
      amount: 0,
      tip: 0,
      address: '1234',
      timeout: null,
      generated: false,
      showcode: false,
      finished: false,
      fiat: true,
    }
  },

  computed: {
    ...mapGetters(['user', 'payreq', 'rate', 'received']),

    code () {
      return this.showcode ? 'Show QR' : 'Show Code'
    }, 

    total () {
      // this.$store.commit('SET_RECEIVED', 0)
      let total = (f(this.amount) + f(this.tip)) / this.rate
      return parseInt(total * 100000000)
    },

    valid () {
      return this.total < 4294967
    },

    currency () {
      if (this.fiat) return this.user.currency
      return 'sats'
    },

    conversion () {
      if (this.fiat) return this.rate
      return parseFloat(1 / this.rate).toFixed(6)
    },
  },

  watch: {
    received () {
      this.finished = false
      this.$nextTick(() => this.$refs.connect ? this.$refs.connect.play() : '')
    },
  },

  methods: {
    ...mapActions(['addInvoice', 'getRates', 'snack', 'clearPayment']),

    symbol (v) {
      if (v === 'sats') return '$'
      return '&#x20BF'
    },

    toggle () {
      this.fiat = !this.fiat 
    },

    async generate () {
      this.$store.commit('SET_RECEIVED', 0)
      this.generated = true
      await this.addInvoice(this.total)
      let canvas = document.getElementById('qr')
      qr.toCanvas(canvas, this.payreq, e => { if (e) console.log(e) })
    },

    finish () {
      this.finished = true
    },
  },

  mounted () {
    this.$store.commit('SET_RECEIVED', 0)
    new Clipboard('.btn')
    this.getRates()
  },
}
</script>

<style lang="stylus" scoped>
  canvas
    position relative
    display block
    height 100%
    margin-left auto
    margin-right auto

  .code
    margin auto
    width 260px
    height 260px
    background #333
    word-wrap break-word
    padding 15px

  [v-cloak]
    display none
</style>
