<template>
  <section>

    <form v-if="pairData.symbol" class="flex-row flex-middle flex-stretch push-bottom" action="#" @submit.prevent="saveAlarm">
      <div class="form-input flex-1 push-right">
        <div class="icon-chart-line push-right"></div>
        <input class="push-right" placeholder="0.00000000" name="price" v-model="curPrice" />
        <div class="text-grey">{{ pairData.asset }}</div>
      </div>
      <button type="submit" class="form-btn bg-success-hover icon-add iconLeft">Set Alarm</button>
    </form>

    <div class="flex-list">
      <div class="flex-header">
        <div class="push-right"><span class="icon-alarm text-faded"></span></div>
        <div class="flex-1 push-right">Symbol</div>
        <div class="flex-1 push-right">Alarm</div>
        <div class="flex-1 push-right">Live Price</div>
        <div class="flex-1 push-right">Status</div>
        <div class="flex-1 push-right">Created</div>
        <div><button class="icon-close text-danger-hover" title="Delete All" @click="flushAlarms" v-tooltip></button></div>
      </div>

      <div v-if="!alarmsList.length" class="flex-item">
        <div class="flex-1 text-info text-faded">
          <span class="icon-info">&nbsp;</span>
          <span v-if="pairData.symbol">There are no alarms for {{ pairData.symbol }}.</span>
          <span v-else>There are no alarms.</span>
        </div>
      </div>

      <div v-for="a in alarmsList" :key="a.id" class="flex-item">
        <div class="push-right">
          <button class="icon-alarm" :class="{ 'text-gain': a.active, 'text-info': !a.active }" title="Toggle" @click="toggleAlarm( a.id, a.symbol, !a.active )" v-tooltip></button>
        </div>
        <div class="flex-1 push-right">
          <button class="text-bright-hover" @click="$bus.emit( 'setRoute', '/symbol/'+ a.symbol )">{{ a.pair }}</button>
        </div>
        <div class="flex-1 push-right">
          <span class="text-big" :class="[ 'text-'+ a.check ]">{{ a.sign }}</span>
          <span class="text-bright">{{ a.price | toFixed( a.asset ) }}</span>
          <span class="text-info">{{ a.asset }}</span>
        </div>
        <div class="flex-1 push-right">
          <span>{{ livePrices[a.symbol] ? livePrices[a.symbol] : 'Loading...' }}</span> <!-- Add this line -->
        </div>
        <div class="flex-1 push-right">
          <span :class="{ 'text-success': a.active, 'text-info': !a.active }">{{ a.active ? 'Active' : 'Triggered' }}</span>
        </div>
        <div class="flex-1 push-right">
          <span class="text-grey">{{ a.time | toDate }}</span>
        </div>
        <div>
          <button class="icon-close" title="Delete" @click="deleteAlarm( a.id, a.symbol )" v-tooltip></button>
        </div>
      </div>

    </div>

  </section>
</template>

<script>
export default {
  props: {
    alarmsData: { type: Array, default() { return [] }, required: true },
    pairData: { type: Object, default() { return {} } },
  },
  data() {
    return {
      curPrice: '',
      livePrices: {} // Add this line
    };
  },
  computed: {
    alarmsList() {
      let list = this.alarmsData.slice();
      let symbol = this.pairData.symbol || '';
      list = this.$utils.sort(list, 'symbol', 'asc');
      if (symbol) {
        list = list.filter(a => a.symbol === symbol);
        list = this.$utils.sort(list, 'active', 'desc');
      }
      this.$emit('listCount', list.length);
      return list;
    },
  },
  methods: {
    saveAlarm(e) {
      let { symbol, asset, close } = this.pairData;
      let price = parseFloat(e.target.price.value) || 0;
      let saved = this.$alarms.add(this.pairData, price);
      if (!saved) return this.$bus.emit('showNotice', 'Please enter a different ' + asset + ' alarm price.', 'warning');
      this.$bus.emit('showNotice', 'New alarm for ' + symbol + ' set for ' + price.toFixed(8) + ' ' + asset + '.', 'success');
    },
    toggleAlarm(id, symbol, toggle) {
      let action = toggle ? 'enabled' : 'disabled';
      this.$alarms.toggle(id, toggle);
      this.$bus.emit('showNotice', 'Alarm for ' + symbol + ' has been ' + action + '.', 'success');
    },
    deleteAlarm(id, symbol) {
      this.$alarms.remove(id);
      this.$bus.emit('showNotice', 'Alarm for ' + symbol + ' has been removed.', 'success');
    },
    flushAlarms() {
      if (!confirm('Delete all alarms from the list?')) return;
      this.$alarms.flush();
      this.$bus.emit('showNotice', 'All alarms have been deleted.', 'success');
    },
    fetchLivePrice(pair) {
      const url = `https://api.binance.com/api/v3/ticker/price?symbol=${pair}`;
      return fetch(url)
        .then(response => response.json())
        .then(data => data.price);
    },
    updateLivePrices() {
      this.alarmsList.forEach(alarm => {
        this.fetchLivePrice(alarm.symbol).then(price => {
          this.$set(this.livePrices, alarm.symbol, price);
        });
      });
    }
  },
  mounted() {
    if (this.pairData.symbol) {
      this.curPrice = Number(this.pairData.close).toFixed(8);
    }
    this.updateLivePrices();
    setInterval(this.updateLivePrices, 30000); // Update every 30 seconds
  }
}
</script>
