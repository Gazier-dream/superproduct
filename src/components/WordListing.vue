<template>
  <div class="row">
    <div class="col-md-3">
      <RouterLink
        class="link"
        :to="{ name: 'word-detail', params: { word } }"
      >
        {{ word }}
      </RouterLink>
    </div>
    <div class="col-md-3">
      <AeAmount
        v-if="buyPrice"
        :amount="buyPrice"
      />
      <Loading
        v-else
        :small="true"
      />
    </div>
    <div class="col-md-3">
      <AeAmount
        v-if="totalSupply !== null && tokenAddress"
        :amount="totalSupply"
        :token="tokenAddress"
      />
      <Loading
        v-else
        :small="true"
      />
    </div>

    <WordBuySellButtons :sale="sale" />
  </div>
</template>

<script>
import TOKEN_SALE_CONTRACT from 'wordbazaar-contracts/TokenSale.aes';
import { mapState } from 'vuex';
import { client, createOrChangeAllowance } from '../utils/aeternity';
import Backend from '../utils/backend';
import { EventBus } from '../utils/eventBus';
import AeAmount from './AeAmount.vue';
import Loading from './Loading.vue';
import { shiftDecimalPlaces } from '../utils';
import WordBuySellButtons from './WordBuySellButtons.vue';

export default {
  name: 'WordListing',
  components: {
    WordBuySellButtons,
    AeAmount,
    Loading,
  },
  props: {
    word: { type: String, required: true },
    sale: { type: String, required: true },
  },
  data: () => ({
    contract: null,
    buyPrice: null,
    sellPrice: null,
    buyAmount: 0,
    sellAmount: 0,
    totalSupply: null,
    tokenAddress: null,
    tokenContract: null,
    showBuyModal: false,
    showSellModal: false,
    loading: true,
  }),
  computed: {
    ...mapState(['address', 'balance', 'tokenBalances']),
    tokenBalance() {
      const balance = this.tokenBalances && this.tokenAddress
        && this.tokenBalances.find((t) => t.token === this.tokenAddress);

      return balance ? balance.balance : '0';
    },
  },
  mounted() {
    this.loadWordData();
  },
  methods: {
    async loadWordData() {
      this.totalSupply = null;
      this.buyPrice = null;

      const data = await Backend.getWordSale(this.sale);
      this.tokenAddress = data.tokenAddress;
      this.totalSupply = data.totalSupply;
      this.buyPrice = data.buyPrice;
      this.sellPrice = data.sellPrice;

      this.loading = false;
      this.buyAmount = 0;
      this.sellAmount = 0;
      this.showSellModal = false;
      this.showBuyModal = false;
    },
    async buy() {
      this.loading = true;

      await this.initContract();
      await this.contract.methods
        .buy({ amount: shiftDecimalPlaces(this.buyAmount, 18).toFixed() });

      await Backend.invalidateTokenCache(this.tokenAddress);
      await Backend.invalidateWordSaleCache(this.sale);
      EventBus.$emit('reloadData');
      this.loadWordData();
    },
    async sell() {
      this.loading = true;

      await this.initContract();
      const amount = shiftDecimalPlaces(this.sellAmount, 18).toFixed();
      await createOrChangeAllowance(this.tokenAddress, amount,
        this.contract.deployInfo.address.replace('ct_', 'ak_'));
      await this.contract.methods.sell(amount);

      await Backend.invalidateWordSaleCache(this.sale);
      EventBus.$emit('reloadData');
      this.loadWordData();
    },
    async initContract() {
      this.contract = this.contract ? this.contract : await client
        .getContractInstance(TOKEN_SALE_CONTRACT, { contractAddress: this.sale });
    },
  },
};
</script>

<style lang="scss" scoped>
.link {
  color: $secondary_color;
  text-decoration: none;
}

h2 {
  margin-top: 1rem;
}

.label {
  color: $light_font_color;
  font-size: 0.7rem;
  margin-bottom: 0.5rem;

  @include desktop {
    margin-bottom: 0.4rem;
    line-height: 0.9rem;
  }
}
</style>
