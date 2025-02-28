<template>
  <v-container fluid :class="{ 'pa-4': !$vuetify.breakpoint.mobile }">
    <alert-disabled system="price" />

    <v-expand-transition>
      <v-app-bar v-if="selected.length > 0" color="blue-grey darken-4" fixed dense>
        <v-row class="px-2" dense justify="end">
          <v-col cols="auto" align-self="center">
            {{ selected.length }} items selected
          </v-col>

          <v-col cols="auto" align-self="center">
            <v-row dense>
              <v-col v-if="selected.length > 0" cols="auto" class="pr-1">
                <batch :length="selected.length" :rules="rules" @save="batchUpdate($event)" />
              </v-col>
              <v-col v-if="selected.length > 0" cols="auto">
                <v-dialog v-model="deleteDialog" max-width="500px">
                  <template #activator="{ on, attrs }">
                    <v-btn class="error" small v-bind="attrs" v-on="on">
                      <v-icon left>
                        mdi-delete-forever
                      </v-icon>
                      Delete
                    </v-btn>
                  </template>

                  <v-card>
                    <v-card-title>
                      <span class="headline">Delete {{ selected.length }} Item(s)?</span>
                    </v-card-title>

                    <v-card-text>
                      <v-data-table
                        dense
                        :items="selected"
                        :headers="headersDelete"
                        :items-per-page="-1"
                        hide-default-header
                        hide-default-footer
                      />
                    </v-card-text>
                    <v-card-actions>
                      <v-spacer />
                      <v-btn text @click="deleteDialog = false">
                        Cancel
                      </v-btn>
                      <v-btn color="error" text @click="deleteSelected">
                        Delete
                      </v-btn>
                    </v-card-actions>
                  </v-card>
                </v-dialog>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-app-bar>
    </v-expand-transition>

    <v-data-table
      v-model="selected"
      calculate-widths
      show-select
      sort-by="command"
      :search="search"
      :loading="state.loading !== ButtonStates.success && state.loadingPrm !== ButtonStates.success"
      :headers="headers"
      :items-per-page="-1"
      :items="items"
      @current-items="saveCurrentItems"
    >
      <template #top>
        <table-search-bar :search.sync="search">
          <edit :rules="rules" @save="refresh()" />
        </table-search-bar>
      </template>

      <template #[`item`]="{ item }">
        <table-mobile :headers="headers" :selected="selected" :item="item" :add-to-selected-item="addToSelectedItem">
          <template #actions>
            <edit :rules="rules" :value="item" @save="refresh()" />
            <v-btn class="danger-hover" icon @click="selected = [item]; deleteDialog = true;">
              <v-icon>
                mdi-delete-forever
              </v-icon>
            </v-btn>
          </template>

          <template #command>
            {{ item.command }}
          </template>

          <template #price>
            <div v-html="priceFormatter(item)" />
          </template>
        </table-mobile>
      </template>
    </v-data-table>
  </v-container>
</template>

<script lang="ts">
import type { PriceInterface } from '@entity/price';
import {
  defineAsyncComponent,
  defineComponent, onMounted, ref, useStore,
} from '@nuxtjs/composition-api';
import { ButtonStates } from '@sogebot/ui-helpers/buttonStates';
import { getLocalizedName } from '@sogebot/ui-helpers/getLocalized';
import { getSocket } from '@sogebot/ui-helpers/socket';
import translate from '@sogebot/ui-helpers/translate';
import axios from 'axios';
import { capitalize, cloneDeep } from 'lodash';

import {
  minLength, minValue, required, startsWith,
} from '~/functions//validators';
import { addToSelectedItem } from '~/functions/addToSelectedItem';
import { error } from '~/functions/error';
import { EventBus } from '~/functions/event-bus';

export default defineComponent({
  components: {
    edit:               defineAsyncComponent(() => import('~/components/manage/price/edit.vue')),
    batch:              defineAsyncComponent(() => import('~/components/manage/price/batch.vue')),
    'table-search-bar': defineAsyncComponent(() => import('~/components/table/searchBar.vue')),
  },
  setup () {
    const store = useStore<any>();
    const search = ref('');
    const items = ref([] as PriceInterface[]);
    const selected = ref([] as PriceInterface[]);
    const selectedItem = ref(null as PriceInterface | null);
    const deleteDialog = ref(false);

    const currentItems = ref([] as PriceInterface[]);
    const saveCurrentItems = (value: PriceInterface[]) => {
      currentItems.value = value;
    };

    const state = ref({ loading: ButtonStates.progress } as {
      loading: number;
    });

    // add oneIsAboveZero validation
    const rules = {
      price:     [minValue(0), required],
      priceBits: [minValue(0), required],
      command:   [required, minLength(2), startsWith(['!'])],
    };

    const headers = [
      { value: 'command', text: translate('command') },
      {
        value: 'enabled', text: translate('enabled'), align: 'center',
      },
      {
        value: 'emitRedeemEvent', text: translate('systems.price.emitRedeemEvent'), align: 'center',
      },
      {
        value: 'price', text: capitalize(translate('systems.price.price.name')), align: 'right',
      },
      { value: 'actions', sortable: false },
    ];
    const headersDelete = [
      { value: 'command', text: translate('command') },
    ];

    onMounted(() => {
      refresh();
    });

    const refresh = () => {
      axios.get('/api/systems/price', { headers: { authorization: `Bearer ${localStorage.accessToken}` } })
        .then(({ data }) => {
          items.value = data.data;
          console.debug(data);

          // we also need to reset selection values
          if (selected.value.length > 0) {
            selected.value.forEach((val, index) => {
              val = items.value.find(o => o.id === val.id) || val;
              selected.value[index] = val;
            });
          }

          state.value.loading = ButtonStates.success;
        }).catch(e => error(e));
    };

    const deleteSelected = async () => {
      deleteDialog.value = false;
      await Promise.all(
        selected.value.map((item) => {
          return new Promise<void>((resolve, reject) => {
            if (!item.id) {
              reject(error('Missing item id'));
              return;
            }
            axios.delete('/api/systems/price/' + item.id, { headers: { authorization: `Bearer ${localStorage.accessToken}` } })
              .finally(() => resolve());
          });
        }),
      );
      refresh();

      EventBus.$emit('snack', 'success', 'Data removed.');
      selected.value = [];
    };

    const isAtLeastOneValueAboveZero = (item: PriceInterface) => {
      return item.price > 0 || item.priceBits > 0;
    };

    function selectItem (item: typeof items.value[number]) {
      selectedItem.value = item;
    }
    function unSelectItem () {
      selectedItem.value = null;
    }
    const priceFormatter = (item: PriceInterface) => {
      const output = [];
      if (item.price !== 0) {
        output.push(`${item.price} ${getLocalizedName(item.price, store.state.configuration.systems.Points.customization.name)}`);
      }
      if (item.priceBits !== 0) {
        output.push(`${item.priceBits} ${getLocalizedName(item.priceBits, translate('bot.bits'))}`);
      }
      return output.join(` ${translate('or')} `);
    };

    const batchUpdate = (value: Record<string, any>) => {
      // check validity
      for (const toUpdate of selected.value) {
        const item = items.value.find(o => o.id === toUpdate.id);
        if (!item) {
          continue;
        }

        let isValid = true;
        for (const key of Object.keys(rules)) {
          for (const rule of (rules as any)[key]) {
            const ruleStatus = rule((toUpdate as any)[key]);
            if (ruleStatus === true) {
              continue;
            } else {
              EventBus.$emit('snack', 'red', `[${key}] - ${ruleStatus}`);
              isValid = false;
            }
          }
        }

        const merged = cloneDeep(item);
        for (const key of Object.keys(value)) {
          if (typeof value[key] !== 'undefined') {
            (merged as any)[key] = value[key];
          }
        }

        if (!isAtLeastOneValueAboveZero(merged)) {
          EventBus.$emit('snack', 'red', `${merged.command} - Points or bits price needs to be set.`);
          isValid = false;
        }

        if (isValid) {
          console.log('Updating', merged);
          getSocket('/admin').emit('systems|price|save', { ...merged }, () => {
            EventBus.$emit('snack', 'success', 'Data updated.');
            refresh();
          });
        }
      }
    };

    return {
      addToSelectedItem: addToSelectedItem(selected, 'id', currentItems),
      saveCurrentItems,
      selectItem,
      unSelectItem,
      selectedItem,
      search,
      items,
      state,
      headers,
      headersDelete,
      getLocalizedName,
      translate,
      isAtLeastOneValueAboveZero,
      priceFormatter,
      refresh,
      batchUpdate,
      capitalize,

      selected,
      deleteDialog,
      rules,
      deleteSelected,
      ButtonStates,
    };
  },
});
</script>
