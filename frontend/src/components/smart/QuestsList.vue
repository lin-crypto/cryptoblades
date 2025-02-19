<template>
  <div v-if="isLoading">
    <i class="fas fa-spinner fa-spin"/>
    {{ $t('quests.loading') }}
  </div>
  <h3 v-else-if="quests.length === 0">
    {{ $t('quests.noQuestTemplatesInSelectedTier') }} </h3>
  <div v-else class="available-quests-container d-flex flex-row gap-3 flex-wrap">
    <div v-for="(quest, index) in quests" :key="quest.id" class="quest-row p-3 gap-5">
      <QuestRequirements :quest="quest" :index="index"/>
      <QuestRewards :quest="quest"/>
      <QuestActions
        :quest="quest" :key="quest.id" :deletable="deletable"
        :questTemplateType="questTemplateType" :showActions="false"
        showSupply @refresh-quest-data="fetchQuests"
      />
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
import {PropType} from 'vue/types/options';
import { Quest } from '@/interfaces';
import { QuestTemplateType } from '@/enums/Quest';
import QuestRequirements from '@/components/smart/QuestRequirements.vue';
import QuestRewards from '@/components/smart/QuestRewards.vue';
import QuestActions from '@/components/smart/QuestActions.vue';
import {mapActions} from 'vuex';

interface StoreMappedActions {
  getQuestTemplates(payload: { tier: number }): Promise<Quest[]>;
}

interface Data {
  quests: Quest[];
  isLoading: boolean;
  isQuestActionLoading: boolean;
}

export default Vue.extend({
  components: {QuestRequirements, QuestRewards, QuestActions},
  props: {
    tier: {
      type: Number as PropType<number>,
      required: true,
    },
    deletable: {
      type: Boolean as PropType<boolean>,
      default: false,
    },
    questTemplateType: {
      type: Number as PropType<QuestTemplateType>,
      required: true,
    },
  },

  data() {
    return {
      quests: [],
      isLoading: false,
      isQuestActionLoading: false,
    } as Data;
  },

  methods: {
    ...mapActions([
      'getQuestTemplates',
    ]) as StoreMappedActions,

    async fetchQuests() {
      try {
        this.isLoading = true;
        this.quests = await this.getQuestTemplates({tier: this.tier});
      } finally {
        this.isLoading = false;
      }
    },
  },

  watch: {
    $props: {
      async handler() {
        await this.fetchQuests();
      },
      deep: true,
      immediate: true,
    },
  },
});
</script>

<style scoped>
.quest-row {
  display: flex;
  width: 45%;
  background: #141414;
  border: 1px solid #60583E;
  border-radius: 10px;
  align-items: center;
}

.available-quests-container {
  width: fit-content;
  justify-content: center;
}

.quest-row {
    flex-direction: column;
  }
@media (max-width: 992px) {
  .quest-row {
    width: 100%;
  }
}
</style>
