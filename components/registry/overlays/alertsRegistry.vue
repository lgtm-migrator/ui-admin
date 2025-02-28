<template>
  <v-expansion-panels v-model="model">
    <slot />
    <v-expansion-panel :readonly="typeof $slots.default === 'undefined'">
      <v-expansion-panel-header>Settings</v-expansion-panel-header>
      <v-expansion-panel-content>
        <v-select
          v-model="options.id"
          :loading="loading"
          :items="itemsOptions"
          label="Alert Registry to use"
        />
      </v-expansion-panel-content>
    </v-expansion-panel>
  </v-expansion-panels>
</template>

<script setup lang="ts">
import type { AlertInterface } from '@entity/alert';
import { isEqual } from 'lodash';

import { setDefaultOpts } from '~/../backend/src/helpers/overlaysDefaultValues';
import GET_ALL from '~/queries/alert/getAll.gql';

const { $graphql } = useNuxtApp();
const props = defineProps({ value: [Object, Array] });
const emit = defineEmits(['input']);

const loading = ref(true);
const items = ref([] as AlertInterface[]);
const refresh = async () => {
  items.value = (await $graphql.default.request(GET_ALL)).alerts;
  loading.value = false;
  setTimeout(() => refresh(), 5000);
};

const itemsOptions = computed(() => {
  return items.value.map(o => ({
    value: o.id,
    text:  o.name,
  }));
});

onMounted(() => {
  refresh();
});

const model = ref(0);
const options = ref(setDefaultOpts(props.value, 'alertsRegistry'));

watch(options, (val: any) => {
  if (!isEqual(props.value, options.value)) {
    emit('input', val);
  }
}, { deep: true, immediate: true });
</script>
