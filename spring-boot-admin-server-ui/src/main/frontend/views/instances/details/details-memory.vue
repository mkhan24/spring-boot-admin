<!--
  - Copyright 2014-2018 the original author or authors.
  -
  - Licensed under the Apache License, Version 2.0 (the "License");
  - you may not use this file except in compliance with the License.
  - You may obtain a copy of the License at
  -
  -     http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
    <sba-panel :title="`Memory: ${name}`" v-if="hasLoaded">
        <div>
            <div v-if="error" class="message is-danger">
                <div class="message-body">
                    <strong>
                        <font-awesome-icon class="has-text-danger" icon="exclamation-triangle"></font-awesome-icon>
                        Fetching memory metrics failed.
                    </strong>
                </div>
            </div>
            <div class="level memory-current" v-if="current">
                <div class="level-item has-text-centered" v-if="current.metaspace">
                    <div>
                        <p class="heading has-bullet has-bullet-primary">Metaspace</p>
                        <p v-text="prettyBytes(current.metaspace)"></p>
                    </div>
                </div>
                <div class="level-item has-text-centered">
                    <div>
                        <p class="heading has-bullet has-bullet-info">Used</p>
                        <p v-text="prettyBytes(current.used)"></p>
                    </div>
                </div>
                <div class="level-item has-text-centered">
                    <div>
                        <p class="heading has-bullet has-bullet-warning">Size</p>
                        <p v-text="prettyBytes(current.committed)"></p>
                    </div>
                </div>
                <div class="level-item has-text-centered" v-if="current.max >= 0">
                    <div>
                        <p class="heading">Max</p>
                        <p v-text="prettyBytes(current.max)"></p>
                    </div>
                </div>
            </div>
            <mem-chart v-if="chartData.length > 0" :data="chartData"></mem-chart>
        </div>
    </sba-panel>
</template>

<script>
  import subscribing from '@/mixins/subscribing';
  import {Observable} from '@/utils/rxjs';
  import moment from 'moment';
  import prettyBytes from 'pretty-bytes';
  import memChart from './mem-chart';

  export default {
    props: ['instance', 'type'],
    mixins: [subscribing],
    components: {memChart},
    data: () => ({
      hasLoaded: false,
      error: null,
      current: null,
      chartData: [],
    }),
    computed: {
      name() {
        switch (this.type) {
          case 'heap':
            return 'Heap';
          case 'nonheap':
            return 'Non heap';
          default:
            return this.type;
        }
      }
    },
    watch: {
      instance() {
        this.subscribe();
      }
    },
    methods: {
      prettyBytes,
      async fetchMetrics() {
        if (this.instance) {
          const responseMax = this.instance.fetchMetric('jvm.memory.max', {area: this.type});
          const responseUsed = this.instance.fetchMetric('jvm.memory.used', {area: this.type});
          const responeMetaspace = this.type === 'nonheap'
            ? this.instance.fetchMetric('jvm.memory.used', {area: this.type, id: 'Metaspace'})
            : null;
          const responseCommitted = this.instance.fetchMetric('jvm.memory.committed', {area: this.type});
          return {
            max: (await responseMax).data.measurements[0].value,
            used: (await responseUsed).data.measurements[0].value,
            metaspace: responeMetaspace ? (await responeMetaspace).data.measurements[0].value : null,
            committed: (await responseCommitted).data.measurements[0].value
          };
        }
      },
      createSubscription() {
        const vm = this;
        if (this.instance) {
          return Observable.timer(0, 2500)
            .concatMap(this.fetchMetrics)
            .subscribe({
              next: data => {
                vm.hasLoaded = true;
                vm.current = data;
                vm.chartData.push({...data, timestamp: moment.now().valueOf()});
              },
              error: error => {
                vm.hasLoaded = true;
                console.warn('Fetching memory metrics failed:', error);
                vm.error = error;
              }
            });
        }
      }
    }
  }
</script>

<style lang="scss">
    @import "~@/assets/css/utilities";

    .memory-current {
        margin-bottom: 0 !important;
    }
</style>