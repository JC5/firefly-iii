<!--
  - MainCreditList.vue
  - Copyright (c) 2020 james@firefly-iii.org
  -
  - This file is part of Firefly III (https://github.com/firefly-iii).
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <https://www.gnu.org/licenses/>.
  -->

<template>
  <div class="card">
    <div class="card-header">
      <h3 class="card-title">{{ $t('firefly.revenue_accounts') }}</h3>
    </div>
    <!-- body if loading -->
    <div v-if="loading && !error" class="card-body">
      <div class="text-center">
        <span class="fas fa-spinner fa-spin"></span>
      </div>
    </div>
    <!-- body if error -->
    <div v-if="error" class="card-body">
      <div class="text-center">
        <span class="fas fa-exclamation-triangle text-danger"></span>
      </div>
    </div>
    <!-- body if normal -->
    <div v-if="!loading && !error" class="card-body table-responsive p-0">
      <table class="table table-sm">
        <caption style="display:none;">{{ $t('firefly.revenue_accounts') }}</caption>
        <thead>
        <tr>
          <th scope="col">{{ $t('firefly.account') }}</th>
          <th scope="col">{{ $t('firefly.earned') }}</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="entry in income">
          <td style="width:20%;"><a :href="'./accounts/show/' +  entry.id">{{ entry.name }}</a></td>
          <td class="align-middle">
            <div v-if="entry.pct > 0" class="progress">
              <div :aria-valuenow="entry.pct" :style="{ width: entry.pct  + '%'}" aria-valuemax="100"
                   aria-valuemin="0" class="progress-bar bg-success"
                   role="progressbar">
                <span v-if="entry.pct > 20">
                  {{ Intl.NumberFormat(locale, {style: 'currency', currency: entry.currency_code}).format(entry.difference_float) }}
                </span>
              </div>
              <span v-if="entry.pct <= 20" style="line-height: 16px;">&nbsp;
              {{ Intl.NumberFormat(locale, {style: 'currency', currency: entry.currency_code}).format(entry.difference_float) }}
              </span>
            </div>
          </td>
        </tr>
        </tbody>
      </table>
    </div>
    <div class="card-footer">
      <a class="btn btn-default button-sm" href="./transactions/deposit"><span class="far fa-money-bill-alt"></span> {{ $t('firefly.go_to_deposits') }}</a>
    </div>
  </div>
</template>

<script>

import {createNamespacedHelpers} from "vuex";
import format from "date-fns/format";

const {mapState, mapGetters, mapActions, mapMutations} = createNamespacedHelpers('dashboard/index')

// See reference nr. 2

export default {
  name: "MainCreditList",
  data() {
    return {
      locale: 'en-US',
      income: [],
      max: 0,
      loading: true,
      error: false
    }
  },
  created() {
    this.locale = localStorage.locale ?? 'en-US';
    this.ready = true;
  },
  computed: {
    ...mapGetters([
                    'start',
                    'end'
                  ]),
    'datesReady': function () {
      return null !== this.start && null !== this.end && this.ready;
    }
  },
  watch: {
    datesReady: function (value) {
      if (true === value) {
        this.getIncome();
      }
    },
    start: function () {
      if (false === this.loading) {
        this.getIncome();
      }
    },
    end: function () {
      if (false === this.loading) {
        this.getIncome();
      }
    },
  },
  methods: {
    getIncome() {
      this.loading = true;
      this.income = [];
      this.error = false;
      // let startStr = this.start.toISOString().split('T')[0];
      // let endStr = this.end.toISOString().split('T')[0];
      let startStr = format(this.start, 'y-MM-dd');
      let endStr = format(this.end, 'y-MM-dd');
      axios.get('./api/v1/insight/income/revenue?start=' + startStr + '&end=' + endStr)
          .then(response => {
            // do something with response.
            this.parseIncome(response.data);
            this.loading = false;
          }).catch(error => {
        this.error = true
      });
    },
    parseIncome(data) {
      for (let i in data) {
        if (data.hasOwnProperty(i)) {
          let mainKey = parseInt(i);
          // contains currency info and entries.
          let current = data[mainKey];
          current.pct = 0;
          this.max = current.difference_float > this.max ? current.difference_float : this.max;
          this.income.push(current);
        }
      }
      if (0 === this.max) {
        this.max = 1;
      }
      // now sort + pct:
      for (let i in this.income) {
        if (this.income.hasOwnProperty(i)) {
          let current = this.income[i];
          current.pct = (current.difference_float / this.max) * 100;
          this.income[i] = current;
        }
      }
      this.income.sort((a,b) => (a.pct > b.pct) ? -1 : ((b.pct > a.pct) ? 1 : 0));
    }
  }
}
</script>
