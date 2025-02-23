<!--
/**
 * This file is part of dHealth Wallet Plugins shared under LGPL-3.0
 * Copyright (C) 2021 Using Blockchain Ltd, Reg No.: 12658136, United Kingdom
 *
 * @package     dHealth Wallet Plugins
 * @subpackage  Health to Earn with Strava
 * @author      Grégory Saive for Using Blockchain Ltd <greg@ubc.digital>
 * @license     LGPL-3.0
 */
-->
<template>
  <div class="link-information-table-container">
    <div class="screen-topbar-container">
      <!-- always displayed -->
      <div class="justified-rows">
        <p>
          Use your dHealth Wallet to earn rewards when you complete activities on Strava&trade;.
          You will be rewarded once per day for any activity completed with a linked Strava account.
        </p>
        <p class="dapp-console">
          This dapp last rewarded a user on the <span class="colored-sec">{{ lastRewardDate }}</span> 
          with <span class="colored-sec">{{ lastRewardAmount }} DHP</span>.
        </p>
      </div>

      <!-- in case account is linked -->
      <div
        v-if="!isLoadingStatus && isAccountLinked"
        class="screen-topbar-inner-container">
        <div class="value-container">
          <div class="status-label"><span>Referral code</span></div>
          <div class="status-value">
            <IconLoading v-if="isLoadingReferral" />
            <div v-else>
              <b>{{ myReferralCode }}</b>
              <ButtonCopy v-model="myReferralCode" />
            </div>
          </div>
        </div>
        <div class="value-container">
          <div class="status-label"><span>Activities</span></div>
          <div class="status-value">
            <IconLoading v-if="isLoadingRewards" />
            <span v-else>{{ countActivities }}</span>
          </div>
        </div>
        <div class="value-container">
          <div class="status-label"><span>Rewards earned</span></div>
          <div class="status-value">
            <IconLoading v-if="isLoadingRewards" />
            <span v-else>{{ totalRewards }} DHP</span>
          </div>
        </div>
        <div class="value-container">
          <div class="status-label"><span>Reload</span></div>
          <div class="status-value">
            <ButtonRefresh @click="onClickRefresh" />
          </div>
        </div>
      </div>

      <!-- in case account is unlinked -->
      <div
        v-else-if="!isLoadingStatus"
        class="screen-topbar-inner-container justified-rows">
        <hr class="separator" />
        <p v-if="!isAccountLinked && hasRewards"
          class="alert-warning mb40">
          This account was previously linked to a Strava account but has been unlinked and is not
          actively linked to a Strava account anymore. You must re-authorize this account to make
          use of your rewards dashboard.
        </p>
        <p v-else-if="!isAccountLinked" class="mb40">
          Authorize our Strava&trade; App <b>dHealth to Earn</b> with your Strava&trade; account by clicking the button below.
          Your preferred browser will open a Strava&trade; address. You will then be asked to Log-In to your Strava account 
          and <i>authorize</i> our App. After having done so, come back here and start earning DHP with your completed Strava&trade; activities.
          If you have a referral code, please enter it in the form input above the Authorize button.
        </p>
      </div>
    </div> <!-- /.screen-topbar-container -->

    <div class="screen-main-container">

      <!-- loading state -->
      <IconLoading
        v-if="isLoadingStatus"
        class="empty-list" />

      <!-- unlinked state -->
      <div
        v-else-if="!isAccountLinked"
        class="autorize-wrapper">
          <div class="display-block mb15">
            <input 
              v-model="friendReferralCode"
              class="referral-input"
              type="text"
              placeholder="Enter referral code" />
            <label>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</label>
          </div>
          <a :href="authorizeUrl" target="_blank" class="authorize-button">Authorize now</a>
          <div class="display-inline ml15">
            <ButtonRefresh @click="onClickRefresh" />
          </div>
      </div>

      <!-- linked state -->
      <div 
        v-else-if="isAccountLinked"
        class="table-wrapper p-custom">
        <GenericTableDisplay
          class="table-section"
          :items="rewardsForAccount"
          :fields="rewardFields"
          :page-size="5"
          :disable-headers="!rewardsForAccount || !rewardsForAccount.length"
          :disable-single-page-links="true"
          :disable-rows-grid="true"
          :disable-placeholder-grid="true"
          :key="rewardsForAccountTimestamp"
        >
          <template v-slot:table-title>
            <h1 class="section-title">
              {{ 'Rewarded activities' }}
            </h1>
          </template>
          <template v-slot:empty>
            <IconLoading v-if="isLoadingRewards" class="empty-list" />
            <h2 v-else class="empty-list">This account has not been rewarded yet.</h2>
          </template>
          <template v-slot:rows="props">
            <GenericTableRow
              v-for="(rowValues, index) in props.items"
              :key="index"
              :row-values="rowValues"
              @click="onClickReward(rowValues.hash)"
            />
          </template>
        </GenericTableDisplay>
      </div>
    </div>

    <ModalRewardViewer
      v-if="showRewardViewerModal"
      :visible="showRewardViewerModal"
      :title="'Congratulation for this reward!'"
      :reward="lastClickedReward"
      :factory="factory"
      @close="showRewardViewerModal = false"
    />
  </div>
</template>

<script lang="ts">
const BigNumber = require('bignumber.js');
const axios = require('axios').default;
const moment = require('moment');
import { Component, Vue, Prop } from 'vue-property-decorator';
import { ButtonCopy, ButtonRefresh, GenericTableDisplay, GenericTableRow, IconLoading } from '@dhealth/wallet-components';
import { Address, RepositoryFactoryHttp } from '@dhealth/sdk';

// internal dependencies
import { RewardDTO, StravaAppService } from '../../../services/StravaAppService';

// internal child components
import ModalRewardViewer from '../../modals/ModalRewardViewer/ModalRewardViewer.vue';

@Component({
  components: {
    ButtonCopy,
    ButtonRefresh,
    GenericTableDisplay,
    GenericTableRow,
    IconLoading,
    ModalRewardViewer,
  },
})
export default class RewardsDashboard extends Vue {
  /**
   * The currently selected account address.
   * @required
   * @var {Address}
   */
  @Prop({ default: undefined })
  public account: Address;

  /**
   * The blockchain network repository factory.
   * @required
   * @var {RepositoryFactoryHttp}
   */
  @Prop({ default: undefined })
  public factory: RepositoryFactoryHttp;

  /**
   * The service handling communication with our
   * firebase cloud functions and rewards.
   * @var {StravaAppService}
   */
  protected stravaApp: StravaAppService;

  /**
   * Whether the component is currently loading
   * an active account's status or not.
   * @var {boolean}
   */
  protected isLoadingStatus: boolean = true;

  /**
   * Whether the component is currently loading
   * an active account's referral code or not.
   * @var {boolean}
   */
  protected isLoadingReferral: boolean = true;

  /**
   * Whether the component is currently loading
   * the last paid out reward or not.
   * @var {boolean}
   */
  protected isLoadingLastReward: boolean = true;

  /**
   * Whether the component is currently loading
   * an account's rewards or not.
   * @var {boolean}
   */
  protected isLoadingRewards: boolean = false;

  /**
   * Whether an account has been linked or not.
   * @var {boolean}
   */
  protected isAccountLinked: boolean = false;

  /**
   * Whether currently displaying the VIEWER modal box
   * @var {boolean}
   */
  public showRewardViewerModal: boolean = false;

  /**
   * The last reward paid out.
   * @var {{amount: number, date: string}}
   */
  protected lastReward: RewardDTO;

  /**
   * The account rewards
   * @var {{amount: number, date: string}[]}
   */
  protected rewardsForAccount: RewardDTO[] = [];

  /**
   * Timestamp of the last update of the rewards.
   * @var {number}
   */
  protected lastUpdatedRewards: number = new Date().valueOf();

  /**
   * The last clicked reward.
   * @var {RewardDTO}
   */
  protected lastClickedReward: RewardDTO;

  /**
   * The referral code of the friend ("referred by").
   * @var {string}
   */
  protected friendReferralCode: string = '';

  /**
   * The user's referral code.
   * @var {string}
   */
  protected myReferralCode: string;

  /**
   * The status poll timeout instance.
   * @var {any}
   */
  protected statusPoll: any;

  /// region computed properties
  /**
   * Getter for fields in the rewards table.
   * @returns {any[]}
   */
  public get rewardFields(): any[] {
    return [
      { name: 'date', label: 'Date' },
      { name: 'height', label: 'Block' },
      { name: 'amount', label: 'Total earned' },
      { name: 'hash', label: 'Transaction hash' }
    ];
  }

  /**
   * Getter for timestamp of last update in the rewards table.
   * @returns {any[]}
   */
  public get rewardsForAccountTimestamp(): number {
    return this.lastUpdatedRewards;
  }

  /**
   * Getter for the authorization URL (Strava app).
   * @returns {string}
   */
  public get authorizeUrl(): string {
    if (! this.account || this.isLoadingStatus) {
      return '#';
    }

    return this.stravaApp.getAuthorizeUrl(this.account, this.friendReferralCode);
  }

  /**
   * Getter for the last activity date.
   * @returns {string}
   */
  public get lastActivityDate(): string {
    if (this.isLoadingRewards || !this.rewardsForAccount.length) {
      return 'N/A';
    }

    return moment(this.rewardsForAccount[0].date).format('Do MMMM YYYY');
  }

  /**
   * Getter for the last activity interval
   * @returns {string}
   */
  public get lastActivityInterval(): string {
    if (this.isLoadingRewards || !this.rewardsForAccount.length) {
      return 'N/A';
    }

    return moment(this.rewardsForAccount[0].date).fromNow();
  }

  /**
   * Getter for the total count of activities.
   * @returns {number}
   */
  public get countActivities(): number {
    if (this.isLoadingRewards || !this.rewardsForAccount.length) {
      return 0;
    }

    return this.rewardsForAccount.length;
  }

  /**
   * Getter for the total amount of rewards received.
   * @returns {number}
   */
  public get totalRewards(): number {
    if (this.isLoadingRewards || !this.rewardsForAccount.length) {
      return 0;
    }

    return this.rewardsForAccount.reduce(
      (acc, curr) => acc.plus(new BigNumber(
        curr.amount
      )),
      new BigNumber('0')
    );
  }

  /**
   * Getter for the last reward amount formatted.
   * @returns {number}
   */
  public get lastRewardAmount(): number {
    if (this.isLoadingLastReward || !this.lastReward) {
      return 0;
    }

    return new BigNumber(
      this.lastReward.amount
    ).shiftedBy(-6); // 6 = divisibility
  }

  /**
   * Getter for the last reward date formatted.
   * @returns {string}
   */
  public get lastRewardDate(): string {
    if (this.isLoadingLastReward || !this.lastReward) {
      return 'N/A';
    }

    return moment(this.lastReward.date).format('Do MMMM YYYY');
  }

  /**
   * Getter for hasRewards property to determine whether
   * an account has received rewards from the dapp.
   * @returns {boolean}
   */
  public get hasRewards(): boolean {
    return !!this.rewardsForAccount && this.rewardsForAccount.length > 0;
  }
  /// end-region computed properties

  /// region components methods
  /**
   * Hook called on creation of the Component (render).
   * @async
   */
  async created() {
    this.stravaApp = new StravaAppService(this.factory)
    await this.refreshData();

    // register status poll until linked or timeout
    if (! this.isAccountLinked) {
      // check status again after 15 seconds
      this.statusPoll = setTimeout(this.refreshData.bind(this), 15000);

      // clear status poll after 10 times
      setInterval(() => {
        clearTimeout(this.statusPoll);
      }, 150000);
    }
    // account is linked, feed referral code
    else {
        this.myReferralCode = await this.stravaApp.getAccountRefCode(this.account);
        this.isLoadingReferral = false;
    }
  }

  /**
   * Hook called on destruction of the Component.
   *
   * @returns {void}
   */
  beforeDestroy() {
    if (!!this.statusPoll) {
      clearTimeout(this.statusPoll);
    }
  }

  /**
   * Hook called on click of a reward in the table.
   *
   * @async
   * @returns {void}
   */
  public onClickReward(hash) {
    const reward: RewardDTO = this.rewardsForAccount.find(
      r => r.hash === hash
    );

    if (!! reward) {
      this.lastClickedReward = reward;
      this.showRewardViewerModal = true;
    }
  }

  /**
   * Hook called on click of the refresh button.
   *
   * @async
   * @returns {void}
   */
  protected async onClickRefresh() {
    await this.refreshData();
    this.$forceUpdate();
  }
  /// end-region components methods

  /// region private api
  /**
   * Private method to refresh data of the component.
   *
   * @async
   * @returns {void}
   */
  private async refreshData() {
    this.isLoadingStatus = !this.isAccountLinked;
    this.isLoadingRewards = true;
    this.isLoadingLastReward = true;

    try {
      // uses health-to-earn firebase backend to find out status
      if (! this.isAccountLinked) {
        this.isAccountLinked = await this.stravaApp.getAccountStatus(this.account);
        this.isLoadingStatus = false;
      }

      // if linked, stop polling
      if (this.isAccountLinked && !!this.statusPoll) {
        clearTimeout(this.statusPoll);
      }

      // if linked, and refcode unknown
      if (this.isAccountLinked && this.isLoadingReferral) {
        this.myReferralCode = await this.stravaApp.getAccountRefCode(this.account);
        this.isLoadingReferral = false;
      }

      // reads transaction of account and formats amounts
      this.rewardsForAccount = (await this.stravaApp.getAccountRewards(this.account)).map(
        r => ({ ...r, amount: new BigNumber(
          r.amount,
        ).shiftedBy(-6), // 6 = divisibility
      }));
      this.isLoadingRewards = false;
      this.lastUpdatedRewards = new Date().valueOf();

      // uses network transactions to find last reward paid out
      this.lastReward = await this.stravaApp.getLastReward();
      this.isLoadingLastReward = false;
    }
    catch (e) {
      console.log(e);
    }
  }
  /// end-region private api
}
</script>

<style lang="less" scoped>
@import "./RewardsDashboard.less";
</style>
