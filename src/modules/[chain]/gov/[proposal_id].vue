<script lang="ts" setup>
import ObjectElement from '@/components/dynamic/ObjectElement.vue';
import {
  useBaseStore,
  useFormatter,
  useGovStore,
  useStakingStore,
  useTxDialog,
} from '@/stores';
import type {
  GovProposal,
  GovVote,
  PaginatedProposalDeposit,
  Pagination,
} from '@/types';
import { ref, reactive } from 'vue';
import Countdown from '@/components/Countdown.vue';
import { computed } from '@vue/reactivity';

const props = defineProps(['proposal_id', 'chain']);
const proposal = ref({} as GovProposal);
const format = useFormatter();
const store = useGovStore();
const dialog = useTxDialog();

store.fetchProposal(props.proposal_id).then((res) => {
  const proposalDetail = reactive(res.proposal);
  // when status under the voting, final_tally_result are no data, should request fetchTally
  if (res.proposal?.status === 'PROPOSAL_STATUS_VOTING_PERIOD') {
    store.fetchTally(props.proposal_id).then((tallRes) => {
      proposalDetail.final_tally_result = tallRes?.tally;
    });
  }
  proposal.value = proposalDetail;
});

const color = computed(() => {
  if (proposal.value.status === 'PROPOSAL_STATUS_PASSED') {
    return 'success';
  } else if (proposal.value.status === 'PROPOSAL_STATUS_REJECTED') {
    return 'error';
  }
  return '';
});
const status = computed(() => {
  if (proposal.value.status) {
    return proposal.value.status.replace('PROPOSAL_STATUS_', '');
  }
  return '';
});

const deposit = ref({} as PaginatedProposalDeposit);
store.fetchProposalDeposits(props.proposal_id).then((x) => (deposit.value = x));

const votes = ref({} as GovVote[]);
const votePage = ref({} as Pagination);
const loading = ref(false);

store.fetchProposalVotes(props.proposal_id).then((x) => {
  votes.value = x.votes;
  votePage.value = x.pagination;
});

function loadMore() {
  if (votePage.value.next_key) {
    loading.value = true;
    store
      .fetchProposalVotes(props.proposal_id, votePage.value.next_key)
      .then((x) => {
        votes.value = votes.value.concat(x.votes);
        votePage.value = x.pagination;
        loading.value = false;
      });
  }
}

function shortTime(v: string) {
  if (v) {
    return format.toDay(v, 'from');
  }
  return '';
}

const votingCountdown = computed((): number => {
  const now = new Date();
  const end = new Date(proposal.value.voting_end_time);
  return end.getTime() - now.getTime();
});

const upgradeCountdown = computed((): number => {
  const height = Number(proposal.value.content?.plan?.height || 0);
  if (height > 0) {
    const base = useBaseStore();
    const current = Number(base.latest?.block?.header?.height || 0);
    return (height - current) * 6 * 1000;
  }
  const now = new Date();
  const end = new Date(proposal.value.content?.plan?.time || '');
  return end.getTime() - now.getTime();
});

const total = computed(() => {
  const tally = proposal.value.final_tally_result;
  let sum = 0;
  if (tally) {
    sum += Number(tally.abstain || 0);
    sum += Number(tally.yes || 0);
    sum += Number(tally.no || 0);
    sum += Number(tally.no_with_veto || 0);
  }
  return sum;
});

const turnout = computed(() => {
  if (total.value > 0) {
    const bonded = useStakingStore().pool?.bonded_tokens || '1';
    return format.percent(total.value / Number(bonded));
  }
  return 0;
});

const yes = computed(() => {
  if (total.value > 0) {
    const yes = proposal.value?.final_tally_result?.yes || 0;
    return format.percent(Number(yes) / total.value);
  }
  return 0;
});

const no = computed(() => {
  if (total.value > 0) {
    const value = proposal.value?.final_tally_result?.no || 0;
    return format.percent(Number(value) / total.value);
  }
  return 0;
});

const veto = computed(() => {
  if (total.value > 0) {
    const value = proposal.value?.final_tally_result?.no_with_veto || 0;
    return format.percent(Number(value) / total.value);
  }
  return 0;
});

const abstain = computed(() => {
  if (total.value > 0) {
    const value = proposal.value?.final_tally_result?.abstain || 0;
    return format.percent(Number(value) / total.value);
  }
  return 0;
});
const processList = computed(() => {
  return [
    { name: 'Turnout', value: turnout.value, class: 'bg-info' },
    { name: 'Yes', value: yes.value, class: 'bg-success' },
    { name: 'No', value: no.value, class: 'bg-error' },
    { name: 'No With Veto', value: veto.value, class: 'bg-red-800' },
    { name: 'Abstain', value: abstain.value, class: 'bg-warning' },
  ];
});
</script>

<template>
  <div>
    <div class="bg-base-100 px-4 pt-3 pb-4 rounded mb-4 shadow">
      <h2 class="card-title flex flex-col md:justify-between md:flex-row">
        <p class="truncate w-full">
          {{ proposal_id }}. {{ proposal.content?.title }}
        </p>
        <div
          class="badge badge-ghost"
          :class="
            color === 'success'
              ? 'text-yes'
              : color === 'error'
              ? 'text-no'
              : 'text-info'
          "
        >
          {{ status }}
        </div>
      </h2>
      <div class="">
        <ObjectElement :value="proposal.content" />
      </div>
    </div>
    <!-- grid lg:grid-cols-3 auto-rows-max-->
    <!-- flex-col lg:flex-row flex -->
    <div class="gap-4 mb-4 grid lg:grid-cols-3 auto-rows-max">
      <!-- flex-1 -->
      <div class="bg-base-100 px-4 pt-3 pb-4 rounded shadow">
        <h2 class="card-title mb-1">Tally</h2>
        <div class="mb-1" v-for="(item, index) of processList" :key="index">
          <label class="block text-sm mb-1">{{ item.name }}</label>
          <div class="h-5 w-full relative">
            <div
              class="absolute inset-x-0 inset-y-0 w-full opacity-10 rounded-sm"
              :class="`${item.class}`"
            ></div>
            <div
              class="absolute inset-x-0 inset-y-0 rounded-sm"
              :class="`${item.class}`"
              :style="`width: ${item.value}`"
            ></div>
            <p
              class="absolute inset-x-0 inset-y-0 text-center text-sm text-[#666] dark:text-[#eee] flex items-center justify-center"
            >
              {{ item.value }}
            </p>
          </div>
        </div>
        <div class="mt-6 grid grid-cols-2">
          <label
            for="vote"
            class="btn btn-primary float-right btn-sm mx-1"
            @click="dialog.open('vote', { proposal_id })"
            >Vote</label
          >
          <label
            for="deposit"
            class="btn btn-primary float-right btn-sm mx-1"
            @click="dialog.open('deposit', { proposal_id })"
            >Deposit</label
          >
        </div>
      </div>

      <div
        class="bg-base-100 px-4 pt-3 pb-5 rounded shadow lg:col-span-2"
      >
        <h2 class="card-title">Timeline</h2>

        <div class="px-1">
          <div class="flex items-center mb-4 mt-2">
            <div class="w-2 h-2 rounded-full bg-error mr-3"></div>
            <div class="text-base flex-1 text-main">
              Submited at: {{ format.toDay(proposal.submit_time) }}
            </div>
            <div class="text-sm">{{ shortTime(proposal.submit_time) }}</div>
          </div>
          <div class="flex items-center mb-4">
            <div class="w-2 h-2 rounded-full bg-primary mr-3"></div>
            <div class="text-base flex-1 text-main">
              Deposited at:
              {{
                format.toDay(
                  proposal.status === 'PROPOSAL_STATUS_DEPOSIT_PERIOD'
                    ? proposal.deposit_end_time
                    : proposal.voting_start_time
                )
              }}
            </div>
            <div class="text-sm">
              {{
                shortTime(
                  proposal.status === 'PROPOSAL_STATUS_DEPOSIT_PERIOD'
                    ? proposal.deposit_end_time
                    : proposal.voting_start_time
                )
              }}
            </div>
          </div>
          <div class="mb-4">
            <div class="flex items-center">
              <div class="w-2 h-2 rounded-full bg-yes mr-3"></div>
              <div class="text-base flex-1 text-main">
                Voting start from {{ format.toDay(proposal.voting_start_time) }}
              </div>
              <div class="text-sm">
                {{ shortTime(proposal.voting_start_time) }}
              </div>
            </div>
            <div class="pl-5 text-sm mt-2">
              <Countdown :time="votingCountdown" />
            </div>
          </div>
          <div>
            <div class="flex items-center mb-1">
              <div class="w-2 h-2 rounded-full bg-success mr-3"></div>
              <div class="text-base flex-1 text-main">
                Voting end {{ format.toDay(proposal.voting_end_time) }}
              </div>
              <div class="text-sm">
                {{ shortTime(proposal.voting_end_time) }}
              </div>
            </div>
            <div class="pl-5 text-sm">
              Current Status: {{ proposal.status }}
            </div>
          </div>

          <div
            class="mt-4"
            v-if="
              proposal?.content?.['@type']?.endsWith('SoftwareUpgradeProposal')
            "
          >
            <div class="flex items-center">
              <div class="w-2 h-2 rounded-full bg-warning mr-3"></div>
              <div class="text-base flex-1 text-main">
                Upgrade Plan:
                <span v-if="Number(proposal.content?.plan?.height || '0') > 0">
                  (EST)</span
                >
                <span v-else>{{
                  format.toDay(proposal.content?.plan?.time)
                }}</span>
              </div>
              <div class="text-sm">
                {{ shortTime(proposal.voting_end_time) }}
              </div>
            </div>
            <div class="pl-5 text-sm mt-2">
              <Countdown :time="upgradeCountdown" />
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="bg-base-100 px-4 pt-3 pb-4 rounded mb-4 shadow">
      <h2 class="card-title">Votes</h2>
      <div class="overflow-x-auto">
        <table class="table w-full table-zebra">
          <tbody>
            <tr v-for="(item, index) of votes" :key="index">
              <td class="py-2 text-sm">{{ item.voter }}</td>
              <td
                class="py-2 text-sm"
                :class="{
                  'text-yes': item.option === 'VOTE_OPTION_YES',
                  'text-gray-400': item.option === 'VOTE_OPTION_ABSTAIN',
                }"
              >
                {{ item.option }}
              </td>
            </tr>
          </tbody>
        </table>

        <button
          @click="loadMore()"
          v-if="votePage.next_key"
          :disabled="loading"
          class="btn btn-outline btn-primary w-full mt-4"
        >
          Load more
        </button>
      </div>
    </div>
  </div>
</template>