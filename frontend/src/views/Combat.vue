<template>
  <div style="display: inline-flex" class="body main-font pl-3">
    <div v-if="ownCharacters.length > 0" :class="equippedWeaponId ? '' : 'combat-disabled'">
      <div v-if="error !== null">
        <div class="col error">{{$t('combat.error')}} {{ error }}</div>
      </div>

      <modal-container
        :noBack="true"
        :modalType="'combat-result'"
        :componentProps="{
          fightResults:fightResults,
          staminaUsed:staminaPerFight,
          isValor: !isGenesisCharacter
        }"/>

      <div class="waitingForResult" v-if="waitingResults">
        <div class="col wa">
            <i class="fas fa-spinner fa-spin fa-1x mr-3"></i>
            {{$t('combat.waiting').toUpperCase()}}
        </div>
      </div>

      <div>
          <div class="col-lg-6 col-md-12 col-xl-6 col-sm-12 adventure">
            <h5 class="m-2">{{$t('combat.adventure')}}</h5>
            <img src="../assets/hint.png" alt="" @click="hideBottomMenu(true)">
          </div>
          <div class="col-lg-12 col-md-12 col-xl-12 col-sm-12 text-right combat-hint" :style="isToggled ? 'display:inline' : 'none'"
           @click="hideBottomMenu(false)">
            <div class="combat-hints">
              <Hint class="mr-3" :text="$t('combat.elementHint')"/>
              <div>
                <span class="fire-icon"/>
                <span class="icon-border">.</span>
              </div> &nbsp;»&nbsp;
              <div>
                <span class="earth-icon" />
                <span class="icon-border">.</span>
              </div> &nbsp;»&nbsp;
              <div>
                <span class="lightning-icon"/>
                <span class="icon-border">.</span>
              </div> &nbsp;»&nbsp;
              <div>
                <span class="water-icon" />
                <span class="icon-border">.</span>
              </div> &nbsp;»&nbsp;
              <div>
                <span class="fire-icon"/>
                <span class="icon-border">.</span>
              </div>
            </div>
          </div>
      </div>

      <img src="../assets/divider7.png" class="info-divider enemy-divider" />

      <div>
        <div class="col">
        <div class="mb-3" :style="'align-self: baseline; width: 20vw'">
          <span class="isMobile label-title">{{$t('combat.selectStamina')}}</span>
          <b-form-select
          class="mt-3 custom-select" v-model="fightMultiplier" :options='setStaminaSelectorValues()' @change="setFightMultiplier()"></b-form-select>
        </div>
          <div  v-if="currentCharacterStamina >= staminaPerFight && equippedWeaponId" class="combat-enemy-container">
            <!-- ------------------------------------------- -->
            <transition-group
                appear @before-enter="beforeEnter" @enter="enter"
                :key="2"
                class="row mb-3 enemy-container" v-if="targets.length > 0">
            <div class="col-12 col-md-6 col-lg-6 col-sm-6 col-xl-3 encounter" v-for="(e, i) in targets" :key="e.original" :data-index="i">
              <div class="encounter-container">
                  <div class="enemy-character" @mouseover="activeCard = i" :disabled="(timeMinutes === 59 && timeSeconds >= 30) ||
                        waitingResults || !charHasStamina()"
                        @click="onClickEncounter(e,i)">

                      <!-- frame -->
                      <div class="frame-line" v-if="activeCard === i">
                        <img style="width: 20rem; height: 35rem;" src="../assets/frame-line-3.png" alt="">
                      </div>

                      <!-- winning chance -->
                      <div class="fight-btn" v-if="activeCard === i">
                        <img src="../assets/fight.png" alt="">
                      </div>

                        <!-- winning chance -->
                      <div class="chance-winning" :style="changeColorChange(getWinChance(e.power, e.trait))">
                        {{ getWinChance(e.power, e.trait).toUpperCase() }}
                        {{$t('combat.victory').toUpperCase()}}
                      </div>

                      <!-- image -->
                      <div class="text-center">
                        <img class="mx-auto enemy-img" :src="getEnemyArt(e.power)" :alt="$t('combat.enemy')" />
                      </div>

                      <!-- trait -->
                      <div class="encounter-element">
                        <span :class="getCharacterTrait(e.trait).toLowerCase() + '-icon'" />
                        <span class="icon-border" style="margin-left: -31.5px" :style="activeCard === i ? 'border:2px solid #9e8a57': ''">.</span>
                      </div>

                      <!-- power -->
                      <div class="encounter-power pt-2">
                          {{$t('combat.power').toUpperCase()}} : {{ e.power.toLocaleString() }}
                      </div>

                      <div class="xp-gain">
                        +{{getPotentialXp(e)}} {{$t('combat.xp')}}
                      </div>

                      <div class="skill-gain mb-1">
                        + ~{{formattedSkill(targetExpectedPayouts[i] * fightMultiplier)}}
                      </div>
                  </div>
              <p v-if="isLoadingTargets">{{$t('loading')}}</p>
              </div>
            </div>
            </transition-group>
          </div>
        </div>
      </div>
      <!-- // error message boxes -->
      <div>
        <div class="col">
          <div class="message-box" v-if="!currentCharacter">{{$t('combat.errors.needToSelectChar')}}</div>
          <div class="row">
            <div class="col-12 text-center">
              <div class="message-box flex-column" v-if="currentCharacter && currentCharacterStamina < staminaPerFight">
                {{$t('combat.needStamina', {staminaPerFight })}}
              </div>
              <div class="message-box" v-if="timeMinutes === 59 && timeSeconds >= 30">{{$t('combat.errors.lastSeconds')}}</div>
              <div class="message-box" v-if="!equippedWeaponId">{{$t('combat.errors.youNeedToHaveWeaponEquippedToCombatYouCanEquipInPlaza')}}</div>
            </div>
          </div>
        </div>
      </div>
      <!-- --------------------------------------- -->
    </div>
    <b-modal class="centered-modal" ref="no-skill-warning-modal" @ok="fightTarget(targetToFight,targetToFightIndex)">
      <template #modal-title>
        <b-icon icon="exclamation-circle" variant="danger"/> {{$t('combat.warning')}}
      </template>
      <span>
        {{$t('combat.youWill1')}} <b> XP </b>! <br>
        {{$t('combat.youWill2')}} <b> {{minutesToNextAllowance}} </b> min. <br>
        {{$t('combat.youWill3')}} <b> {{lastAllowanceSkill}} </b>  {{$t('combat.youWill4')}}
      </span>
    </b-modal>
    <div class="blank-slate" v-if="ownCharacters.length === 0">
      <div v-if="ownCharacters.length === 0">{{$t('combat.noCharacters')}}</div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
import {getEnemyArt} from '../enemy-art';
import {CharacterTrait, ICharacter, IPowerData, ITarget} from '../interfaces';
import Hint from '../components/Hint.vue';
import {fromWeiEther, toBN} from '../utils/common';
import BigNumber from 'bignumber.js';
import ModalContainer from '../components/modals/ModalContainer.vue';
import {mapActions, mapGetters, mapMutations, mapState} from 'vuex';
import gasp from 'gsap';
import i18n from '../i18n';
import { Accessors } from 'vue/types/options';

interface StoreMappedCombatActions {
  fetchCharacterStamina(characterId: number): Promise<void>;
  fetchTargets(
    { characterId }:
    { characterId: number }): Promise<void>;
  doEncounterPayNative(
    { characterId,
      targetString,
      fightMultiplier,
      offsetCost }:
    { characterId: number,
      targetString: number,
      fightMultiplier: number,
      offsetCost: BigNumber
    }): Promise<{
    isVictory: boolean,
    playerRoll: string,
    enemyRoll: string,
    xpGain: any,
    skillGain: any,
    bnbGasUsed: string,
  }>;
  fetchFightRewardSkill(): Promise<string>;
  fetchFightRewardValor(): Promise<string>;
  fetchFightRewardXp(): Promise<string[][]>;
  fetchExpectedPayoutForMonsterPower(
    { power }:
    { power: stringOrNumber }): Promise<string>;
  fetchHourlyAllowance(): Promise<string>;
  fetchHourlyPowerAverage(): Promise<string>;
  fetchHourlyPayPerFight(): Promise<string>;
  getCurrentSkillPrice(): Promise<string>;
  getNativeTokenPriceInUsd(): Promise<string>;
  getCombatTokenChargePercent(): Promise<string>;
  getFightXpGain(): Promise<number>;
}

interface StoreMappedActions{
  fetchCharacterWeapon(characterId: string | number): Promise<number>;
}

interface StoreMappedState {
  currentCharacterId: number,
  characters: ICharacter[],
}

interface StoreMappedGetters {
  ownCharacters: ICharacter[],
  currentCharacter: ICharacter,
  currentCharacterStamina: number,
  getPowerData(characterId: number): IPowerData,
}

interface StoreMappedCombatMutations {
  setIsInCombat(isInCombat: boolean): boolean,
}

interface StoreMappedCombatGetters {
  getTargetsByCharacterId(currentCharacterId: number): ITarget | any[],
  fightGasOffset: string,
  fightBaseline: string,
}

type stringOrNull = string | null;
type numberOrNull = number | null;
type stringOrNumber = string | number;

interface ICombatData {
  powerData: IPowerData;
  error: stringOrNull;
  waitingResults: boolean;
  resultsAvailable: boolean;
  fightResults: any;
  intervalSeconds: any;
  intervalMinutes: any;
  timeSeconds: any;
  timeMinutes: any;
  fightXpGain: number;
  fightMultiplier: number;
  staminaPerFight: number;
  targetExpectedPayouts: string[];
  targetToFight: any;
  targetToFightIndex: any;
  minutesToNextAllowance: any;
  secondsToNextAllowance: any;
  lastAllowanceSkill: stringOrNull;
  nextAllowanceCounter: any;
  powerAvg: stringOrNull;
  expectedSkill: stringOrNull;
  activeCard: any;
  isToggled: boolean;
  gridStyling: string;
  counterInterval: any;
  equippedWeaponId: string | number;
}

export default Vue.extend({
  data() {
    return {
      powerData: {},
      error: null,
      waitingResults: false,
      resultsAvailable: false,
      fightResults: null,
      intervalSeconds: null,
      intervalMinutes: null,
      timeSeconds: null,
      timeMinutes: null,
      fightXpGain: 32,
      fightMultiplier: Number(localStorage.getItem('fightMultiplier')),
      staminaPerFight: 40,
      targetExpectedPayouts: new Array(4),
      targetToFight: null,
      targetToFightIndex: null,
      minutesToNextAllowance: null,
      secondsToNextAllowance: null,
      lastAllowanceSkill: null,
      nextAllowanceCounter: null,
      powerAvg: null,
      expectedSkill: null,
      activeCard: null,
      isToggled: false,
      gridStyling:'justify-content:flex-start; gap:2.5vw',
      counterInterval: null,
      equippedWeaponId: '',
    } as ICombatData;
  },
  async mounted() {
    this.powerData = this.getPowerData(this.currentCharacterId);
    this.fightXpGain = await this.getFightXpGain();
    this.equippedWeaponId = await this.fetchCharacterWeapon(this.currentCharacterId);
    if(this.currentCharacterId !== null && this.currentCharacterId !== undefined) {
      await this.fetchTargets({ characterId: this.currentCharacterId });
    }
  },
  created() {
    this.intervalSeconds = setInterval(() => (this.timeSeconds = new Date().getSeconds()), 5000);
    this.intervalMinutes = setInterval(() => (this.timeMinutes = new Date().getMinutes()), 20000);
    this.updateStaminaPerFight();
    this.counterInterval = setInterval(async () => {
      await this.getExpectedPayout();
    }, 1000);

  },
  beforeDestroy() {
    clearInterval(this.intervalSeconds);
    clearInterval(this.intervalMinutes);
    clearInterval(this.counterInterval);
  },
  computed: {
    ...(mapState([
      'currentCharacterId',
      'characters',
    ]) as Accessors<StoreMappedState>),
    ...(mapGetters([
      'ownCharacters',
      'currentCharacter',
      'currentCharacterStamina',
      'getPowerData',
    ]) as Accessors<StoreMappedGetters>),
    ...(mapGetters('combat', [
      'getTargetsByCharacterId',
      'fightGasOffset',
      'fightBaseline',
    ]) as Accessors<StoreMappedCombatGetters>),

    targets(): any[] | ITarget {
      return this.getTargetsByCharacterId(this.currentCharacterId);
    },

    isLoadingTargets(): boolean {
      const targets = this.targets as any[];
      return targets.length === 0 && !!this.currentCharacterId;
    },

    selections(): any[] {
      return [this.currentCharacterId];
    },

    updateResults(): any[] {
      return [this.fightResults, this.error];
    },

    isGenesisCharacter(): boolean {
      return this.currentCharacter.version === 0;
    },

    selectedCharacter(): ICharacter{
      return this.characters[this.currentCharacterId];
    }
  },

  watch: {
    async selections([characterId]) {
      this.powerData = this.getPowerData(characterId);
      if(this.powerData) {
        await this.fetchTargets({ characterId });
      }
    },

    async targets() {
      await this.getExpectedPayouts();
    },

    async updateResults([fightResults, error]) {
      this.resultsAvailable = fightResults !== null;
      this.waitingResults = fightResults === null && error === null;
      this.setIsInCombat(this.waitingResults);
      if (this.resultsAvailable && error === null) (this as any).$bvModal.show('modal-info');
    },

    async selectedCharacter(newValue){
      if (newValue) {
        this.equippedWeaponId = await this.fetchCharacterWeapon(this.currentCharacterId);
      }
    },
  },

  methods: {
    ...(mapActions('combat',
      [
        'fetchTargets',
        'doEncounterPayNative',
        'fetchFightRewardSkill',
        'fetchFightRewardValor',
        'fetchFightRewardXp',
        'fetchExpectedPayoutForMonsterPower',
        'fetchHourlyAllowance',
        'fetchHourlyPowerAverage',
        'fetchHourlyPayPerFight',
        'getCurrentSkillPrice',
        'getNativeTokenPriceInUsd',
        'getCombatTokenChargePercent',
        'fetchCharacterStamina',
        'getFightXpGain',
      ]) as StoreMappedCombatActions),
    ...(mapMutations('combat', ['setIsInCombat']) as StoreMappedCombatMutations),
    ...mapActions(['fetchCharacterWeapon']) as StoreMappedActions,
    getEnemyArt,
    charHasStamina(){
      return this.currentCharacterStamina >= this.staminaPerFight;
    },
    getCharacterTrait(trait: CharacterTrait) {
      return CharacterTrait[trait];
    },
    getWinChance(enemyPower: number, enemyElement: number) {
      const totalMultipliedPower = this.powerData.pvePower[enemyElement];
      const playerMin = totalMultipliedPower * 0.9;
      const playerMax = totalMultipliedPower * 1.1;
      const playerRange = playerMax - playerMin;
      const enemyMin = enemyPower * 0.9;
      const enemyMax = enemyPower * 1.1;
      const enemyRange = enemyMax - enemyMin;
      let rollingTotal = 0;
      // shortcut: if it is impossible for one side to win, just say so
      if (playerMin > enemyMax) return i18n.t('combat.winChances.veryLikely');
      if (playerMax < enemyMin) i18n.t('combat.winChances.unlikely');

      // case 1: player power is higher than enemy power
      if (playerMin >= enemyMin) {
        // case 1: enemy roll is lower than player's minimum
        rollingTotal = (playerMin - enemyMin) / enemyRange;
        // case 2: 1 is not true, and player roll is higher than enemy maximum
        rollingTotal += (1 - rollingTotal) * ((playerMax - enemyMax) / playerRange);
        // case 3: 1 and 2 are not true, both values are in the overlap range. Since values are basically continuous, we assume 50%
        rollingTotal += (1 - rollingTotal) * 0.5;
      } // otherwise, enemy power is higher
      else {
        // case 1: player rolls below enemy minimum
        rollingTotal = (enemyMin - playerMin) / playerRange;
        // case 2: enemy rolls above player maximum
        rollingTotal += (1 - rollingTotal) * ((enemyMax - playerMax) / enemyRange);
        // case 3: 1 and 2 are not true, both values are in the overlap range
        rollingTotal += (1 - rollingTotal) * 0.5;
        //since this is chance the enemy wins, we negate it
        rollingTotal = 1 - rollingTotal;
      }
      if (rollingTotal <= 0.3) return i18n.t('combat.winChances.unlikely');
      if (rollingTotal <= 0.5) return i18n.t('combat.winChances.possible');
      if (rollingTotal <= 0.7) return i18n.t('combat.winChances.likely');
      return i18n.t('combat.winChances.veryLikely');
    },
    getElementAdvantage(playerElement: number, enemyElement: number) {
      if ((playerElement + 1) % 4 === enemyElement) return 1;
      if ((enemyElement + 1) % 4 === playerElement) return -1;
      return 0;
    },

    // for enemy card animaton
    enter(el: HTMLElement, done: any) {
      gasp.to(el, {
        opacity: 1,
        y: 0,
        duration: 0.5,
        onComplete: done,
        delay: el.dataset.index as any * 0.1
      });
    },
    beforeEnter(el: HTMLElement){
      el.style.opacity = '0';
      el.style.transform = 'translateY(100px)';
    },
    // -------------------

    hideBottomMenu(bol: boolean){
      this.isToggled = bol;
    },

    async getExpectedPayout() {
      this.powerAvg = await this.fetchHourlyPowerAverage();
      this.expectedSkill = fromWeiEther(await this.fetchHourlyPayPerFight());
    },
    async getHourlyAllowance(){
      const fetchHourlyAllowance = await this.fetchHourlyAllowance();
      this.lastAllowanceSkill = this.formattedSkill(fetchHourlyAllowance);
    },
    async onClickEncounter(targetToFight: ITarget, targetIndex: number) {
      // this.$bvModal.show('waitingForResult');
      if(this.targetExpectedPayouts[targetIndex] === '0'){
        this.targetToFight = targetToFight;
        this.targetToFightIndex = targetIndex;
        await this.getHourlyAllowance();
        (this.$refs['no-skill-warning-modal'] as any).show();
      }
      else{
        this.fightTarget(targetToFight, targetIndex);
      }
    },

    async fightTarget(targetToFight: ITarget, targetIndex: number){
      if (this.currentCharacterId === null) {
        return;
      }
      this.waitingResults = true;

      // Force a quick refresh of targets
      await this.fetchTargets({ characterId: this.currentCharacterId });
      // If the targets list no longer contains the chosen target, return so a new target can be chosen
      const targets = this.targets as any[];
      if (targets[targetIndex].original !== targetToFight.original) {
        this.waitingResults = false;
        return;
      }

      this.fightResults = null;
      this.error = null;
      this.setIsInCombat(this.waitingResults);
      try {
        const targetPower = targetToFight.power;
        const expectedPayoutWei = new BigNumber(await this.fetchExpectedPayoutForMonsterPower({ power: targetPower }));

        const nativeTokenPriceUsd = new BigNumber(await this.getNativeTokenPriceInUsd());
        const skillPriceUsd = new BigNumber(await this.getCurrentSkillPrice());
        const tokenChargePercentage = (await this.getCombatTokenChargePercent());

        const offsetToPayInNativeToken = (
          expectedPayoutWei.multipliedBy(tokenChargePercentage).div(100).multipliedBy(skillPriceUsd.gt(0) ? skillPriceUsd : 1)
        ).div(nativeTokenPriceUsd.gt(0) ? nativeTokenPriceUsd : 1).integerValue(BigNumber.ROUND_DOWN);

        this.fightResults = await this.doEncounterPayNative({
          characterId: this.currentCharacterId,
          targetString: targetIndex,
          fightMultiplier: this.fightMultiplier,
          offsetCost: offsetToPayInNativeToken
        });

        await this.fetchFightRewardSkill();
        await this.fetchFightRewardValor();
        await this.fetchFightRewardXp();

        await this.fetchCharacterStamina(this.currentCharacterId);

        this.error = null;
      } catch (error: any) {
        console.error(error);
        this.error = error.message;
      }
    },

    formattedSkill(skill: stringOrNumber) {
      const skillBalance = fromWeiEther(skill.toString());
      return `${toBN(skillBalance).toFixed(6)} ${this.isGenesisCharacter ? 'SKILL' : 'VALOR'}`;
    },

    getPotentialXp(targetToFight: ITarget) {
      const totalPower = this.powerData.pvePower[4]; // using base power
      //Formula taken from getXpGainForFight funtion of cryptoblades.sol
      return Math.floor((targetToFight.power / totalPower) * this.fightXpGain) * this.fightMultiplier;
    },

    setFightMultiplier() {
      localStorage.setItem('fightMultiplier', this.fightMultiplier.toString());
      this.updateStaminaPerFight();
    },

    setStaminaSelectorValues() {
      if(this.currentCharacterStamina < 40) {
        return [{ value: this.fightMultiplier, text: i18n.t('combat.moreStamina').toString(), disabled: true}];
      }

      const choices: {value: numberOrNull, text: string, disabled?: boolean }[] = [
        {value: null, text: i18n.t('combat.pleaseSelect').toString(), disabled: true},
      ];

      const addChoices = [];

      if(this.currentCharacterStamina >= 200) {
        addChoices.push({ value: 5, text: '200' });
      }

      if(this.currentCharacterStamina >= 160) {
        addChoices.push({ value: 4, text: '160' });
      }

      if(this.currentCharacterStamina >= 120) {
        addChoices.push({ value: 3, text: '120' });
      }

      if(this.currentCharacterStamina >= 80) {
        addChoices.push({ value: 2, text: '80' });
      }

      if(this.currentCharacterStamina >= 40) {
        addChoices.push({ value: 1, text: '40' });
      }

      choices.push(...addChoices.reverse());

      return choices;
    },

    async getExpectedPayouts() {
      if(!this.targets) return;
      const expectedPayouts = new Array(4);
      const targets = this.targets as ITarget[];
      for(let i = 0; i < targets.length; i++) {
        const expectedPayout = await this.fetchExpectedPayoutForMonsterPower({ power: targets[i].power });
        expectedPayouts[i] = expectedPayout;
      }
      this.targetExpectedPayouts = expectedPayouts;
    },

    updateStaminaPerFight() {
      this.staminaPerFight = 40 * Number(localStorage.getItem('fightMultiplier'));
    },

    changeColorChange(stat: string){
      let bgColor;
      if(stat.toUpperCase() === 'UNLIKELY'){
        bgColor =  'background-color: #B10000 !important';
      }else if(stat.toUpperCase() === 'VERY LIKELY'){
        bgColor =  'background-color: #7BA224 !important';
      }else if(stat.toUpperCase() === 'POSSIBLE'){
        bgColor =  'background-color: #D16100 !important';
      }else{
        bgColor = 'background-color: #ff7700 !important';
      }
      return bgColor;
    }
  },

  components: {
    Hint,
    ModalContainer
  },
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Cardo:ital,wght@0,400;0,700;1,400&display=swap');
.custom-select{
  background-color:#010D22;
  color:#fff
}
.body{
  background: linear-gradient(0deg, rgba(0, 14, 41, 0.68), rgba(0, 14, 41, 0.68)), url('../assets/combat-bg.png');
  background-size: clamp(100%, 100%, 100%) auto;
  background-repeat: no-repeat;
  min-height: 100%;
}
h5{
  font-family: 'Trajan', serif;
  font-size: 25px;
  font-weight: 400;
}

.label-title{
  font-family: Oswald;
  color: #fff;
  text-transform: uppercase;
  font-size: 20px;
}


.enemy-character {
  position: relative;
  width: inherit;
  cursor: pointer;
  background-position: center;
  background-repeat: no-repeat;
  background-size: clamp(100%, 100%, 100%) auto;
  background-image: url('../assets/enemy-bg-transparent.png');
  background-color: linear-gradient(45deg, rgba(20, 20, 20, 1) 100%, #242720 100%);
  border: 1px solid #a28d54;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  margin-bottom: 10px;
  box-shadow: 0px 6px 8px rgba(0, 0, 0, 0.705), 0px 12px 7px rgba(0, 0, 0, 0.5), 0px 9px 12px rgba(0, 0, 0, 0.1);
}

.enemy-character:hover{
  transition: all 0.2s ease-in-out;
  border: 1px solid #a28d54;
  box-shadow: inset 0px 0px 0px 0px rgba(0,0,0,0.6);
}

.encounter img {
  width: 13rem;
  transition: 1s all;
}

.payout-info {
  margin: auto;
  text-align: center;
  padding-top: 1em;
  font-size: 1.5em;
  display: flex;
  justify-content: center;
  align-items: center;
}

.frame-line{
  position: absolute;
  display: flex;
  justify-content: center;
}


.frame-line:hover{
  max-width: 112%;
  opacity: 1;
}


@keyframes resizeUp {
  0%   {
    width: 80%;
    opacity: 0;
  }
  100%  {
    width: 115%;
    opacity: 1;
  }
}


.combat-hints {
  margin: auto;
  text-align: center;
  font-size: 2em;
  display: flex;
  align-items: center;
  justify-content: center;
}

.combat-hints > div{
  display: flex;
  padding-right: 12px;
  padding-left: 12px;
}

.combat-hints > div > .fire-icon,
.combat-hints > div > .earth-icon,
.combat-hints > div > .lightning-icon,
.combat-hints > div > .water-icon{
  max-height: 25px !important;
  max-width: 25px !important;
  width: auto;
  height: auto;
}

.combat-hints .hint {
  margin-left: 50px;
  width: 30px;
}

.waiting {
  font-size: 2em;
  margin: auto;
  text-align: center;
}

.header-row {
  display: flex;
  align-items: center;
}

.header-row h1 {
  margin-left: 10px;
  margin-bottom: 5px;
}

.header-row .hint {
  font-size: 2em;
}

.message-box {
  font-family: Trajan;
  display: flex;
  justify-content: center;
  width: 100%;
  font-size: 2em;
  height: calc(80vh - 150px);
}

div.encounter.text-center {
  flex-basis: auto !important;
}

.encounter-container {
  position: relative;
}

.encounter {
  display: flex;
  justify-content: center;
}

.xp-gain,
.skill-gain {
  color: rgb(158, 138, 87) !important;
  font-family: Roboto;
  font-size: 12px;
}

.encounter-power{
  color: #fff;
  font-size: 1.3rem;
  font-weight: 600;
  margin-top: 1.5rem;
  font-family: Oswald;
}

.chance-winning{
    z-index: 1;
    color: #fff;
    padding: 1px 15px;
    border-radius: 2px;
    font-size: 13px;
    font-family: Oswald;
    text-transform: capitalize;
    position: absolute;
    top: 15px;
    right: 15px;
}

.fight-btn> img{
  width: 22px !important;
}

.fight-btn{
  position:absolute;
  top: 15px;
  left: 15px;
  animation-name: moveUpFade;
  animation-duration: 1s;
}

.modal-body{
  background-image: url('../assets/enemy-bg.png') !important;
}

@keyframes moveUpFade {
  0%   {
    margin-top: 20px;
    opacity: 0;
  }
  100%  {
    margin-top: 10px;
    opacity: 1;
  }
}



.encounter-element {
  top: 25px;
  font-size: 30px;
}

.encounter-element > .icon-border{
  width: 33px;
  height: 33px;
  margin-top: 1px
}

.encounter-power {
  bottom: 60px;
}

.xp-gain {
  bottom: 40px;
  font-size: 1rem;
}

.skill-gain {
  bottom: 20px;
  font-size: 1rem;
}

.victory-chance {
  left: 0;
  right: 0;
  text-align: center;
  font-size: 1.5em;
  text-shadow: -1px 0 #000, 0 1px #000, 1px 0 #000, 0 -1px #000;
}

/* Mobile Support Classes*/
.mobile-divider-wrapper {
  width: 100%;
  display: flex;
}

.mobile-divider {
  margin: auto;
}

.combat-enemy-container {
  display: flex;
  flex-direction: column;
  margin-bottom: 50px;
}

.enemy-container {
  flex: 3;
}

.enemy-list {
  display: flex;
  flex-wrap: wrap;
  padding-left: 30px;
  padding-right: 30px;
}

.enemy-energy {
  top: -30px;
  position: relative;
}


.hideMobile{
  display: none;
}

h1 {
  font-weight: 900 !important;
  text-align: center;
  font-size: 3vw;
  padding-top: 0px;
}

.encounter-button {
  display: block;
  margin: 0 auto;
  height: 5em;
  width: 13em;
  position: relative;
  top: 3vw;
  margin-top: 2em;
}

.enemy-img {
  position: relative;
  top: -30px;
}

.adventure{
    text-align: left;
    margin-top: 30px;
  }

.btn-trigger{
  display: none;
}

@media (max-width: 1334px) {
  .enemy-list {
    flex-flow: row wrap;
    align-items: center;
  }
  .enemy-list > .enemy-list-child{
     flex-basis: 50%;
  }
  .encounter-button {
    margin-top: 1.35em;
  }

}

.message-box  .ct-btn > img{
  width: 30px;
  margin-left: 20px;
}


.message-box  .ct-btn{
  margin-left: 20px;
  background-color: rgba(255, 255, 255, 0);
}

.message-box  .ct-btn:hover{
  margin-top: -10px;
  transition: 0.3s all ease-in-out;
}

.showInMobile{
  display: none;
}

.adventure > img{
  display: none;
}


@media all and (max-width: 600px) {
  .combat-hints > div > .fire-icon,
  .combat-hints > div > .earth-icon,
  .combat-hints > div > .lightning-icon,
  .combat-hints > div > .water-icon {
    max-height: 20px !important;
    max-width: 20px !important;
    width: auto;
    height: auto;
  }

  .hideMobile{
    display: inline;
  }

  .body  > div{
    padding-left: 0px;
  }

  .isMobile{
    display: none;
  }

  .combat-hints > div > .icon-border{
    height: 21px !important;
    width: 21px !important;
  }

  .showInMobile{
    display: inline;
    font-family: Roboto;
    font-size: 11px;
  }

  .adventure{
    display: flex;
    margin-right: 10px;
    justify-content: space-between;
    margin-top: 10px;
  }

  .adventure > h5{
    margin-left: 10px;
  }

  .adventure > img{
    display: inline;
    width: 35px;
    height: 35px;
  }

  .combat-hint{
    display: none;
    z-index: 99;
    background-color:rgba(1, 13, 34,1);
    transition: all 1s ease-in-out;
  }

  .enemy-container {
    flex: none;
    overflow-x: hidden;
    height: 60vh;
  }
  .header-row {
    justify-content: center !important;
  }

  .enemy-character {
    width: 20rem;
  }

  .hideMenu{
    margin-bottom: -120px;
    transition: all 1s ease-in-out;
  }

  .showMenu{
    margin-bottom: 0px;
    transition: all 1s ease-in-out;
  }

  .btn-trigger{
    right: 30px;
    position: absolute;
    width: 20px;
    z-index: 100;
  }

  .btn-trigger{
    display: inline;
  }

  .btn-trigger > img{
    width: 30px;
    margin-left: 0px;
    margin-top: -20px;
    margin-top: -70px;
  }

  .rotateUp{
    transform: rotate(270deg);
    transition: all 1s ease-in-out;
  }

  .rotateDown{
    transform: rotate(90deg);
    transition: all 1s ease-in-out;
  }

  .waitingForResult .col{
    font-family: Trajan;
    font-size: 15px !important;
  }

  .combat-hint{
    position: absolute;
  }

}

/* Needed to asjust image size, not just image column-size and other classes to accommodate that */
@media all and (max-width: 767.98px) {
  .encounter img {
    max-width: 140px;
  }

  .frame-line {
    display: none;
  }

  .enemy-list{
    flex-direction:column;
    align-items:center;
  }
  .combat-enemy-container {
    flex-direction: column;
    align-items: center;
  }
  .results-panel {
    width: 100%;
  }
}
.hint.has-tooltip {
  font-size: 1.8rem;
  display: inline-block;
  margin-left: 10px;
}
.dark-bg-text {
  width: 100% !important;
}
.content {
  padding: 0 !important;
}

.encounter-container {
  margin-bottom: 50px;
}

#gtag-link-others {
  margin: 0 auto;
  display: block;
  position: relative;
  margin-top: 20px;
  width: 100%;
}
.ml-3 {
  margin-left: 0px !important;
}
.header-row {
  display: block;
  text-align: center;
}
#expectedSkillHint{
  margin:0;
  font-size: 1em;
}

.cw-content h4{
    font-family: 'Trajan', serif;
    text-transform: capitalize;
}


.cw-content{
  padding: 40px;
  border: 1px solid #ffffff52;
}
/* --------------------------------- */
.ct-btn{
  padding: 0px;
  height: fit-content;
  border: none !important;
}

.ct-btn:hover{
  border: none !important;
}


.icon-border{
    border: 1px solid #9e8a57;
    height: 27px;
    width: 27px;
    color: #f0f8ff00;
    position: absolute;
    transform: rotate(45deg);
    margin-left: -1px;
    margin-top: -1px;
}


/* enemy card animaton */
.slide-fade-enter-active {
  transition: all 3s ease-in-out;
}

.slide-fade-leave-active {
  transition: all 3s ease-in-out;
  /* transition: all 0.4s cubic-bezier(1, 0.5, 0.8, 1); */
}

.slide-fade-enter-from,
.slide-fade-leave-to {
  transform: translateY(50px);
  opacity: 0;
}

.enemy-name{
  margin-top: 10px;
  font-family: 'Trajan', 'serif';
  font-weight: 600;
}

.waitingForResult{
  height: 90vh;
  position: absolute;
  width: 99%;
  z-index: 98;
}

.waitingForResult .col{
  font-family: Trajan;
    font-size: 25px;
    position: absolute;
    text-align: center;
    bottom: 40vh;
    background: linear-gradient(to right, rgba(255,0,0,0), rgb(0 0 0),rgb(6 0 0 / 0%));
    z-index: 99;
    padding: 20px;
}

.footer-close{
  margin: auto;
}

.footer-close > span{
  cursor: pointer;
}

.footer-close > .tap{
  font-size: 15px;
  color: #fff;
  margin-top: 40px;
  text-align: center;
  margin-left: auto;
  margin-bottom: 20px;
  font-family: Roboto;
}

.vertical-decoration.bottom{
  transform: rotate(0deg) !important;
}

.combat-disabled {
  pointer-events: none;
  opacity: 0.5;
}

@media (max-width: 575.98px) {
  .show-reforged {
    width: 100%;
    justify-content: center;
    display: block;
  }
}
</style>
