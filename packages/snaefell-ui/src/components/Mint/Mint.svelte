<script lang="ts">
  import { ResponsiveController } from '@taiko/ui-lib';
  import { getAccount } from '@wagmi/core';
  import { getContext } from 'svelte';
  import { t } from 'svelte-i18n';
  import { zeroAddress } from 'viem';

  import { Divider } from '$components/core/Divider';
  import InfoRow from '$components/core/InfoRow/InfoRow.svelte';
  import { ProgressBar } from '$components/core/ProgressBar';
  import User from '$lib/user';
  import { classNames } from '$lib/util/classNames';
  import type { IMint } from '$stores/mint';
  import { connectedSourceChain } from '$stores/network';
  import { Button } from '$ui/Button';
  import { Spinner } from '$ui/Spinner';

  import Token from '../../lib/token';
  import getConfig from '../../lib/wagmi/getConfig';
  import type { IAddress } from '../../types';
  import { NftRenderer } from '../NftRenderer';
  import {
    counterClasses,
    currentMintedClasses,
    eligibilityLabelClasses,
    eligibilityValueClasses,
    infoRowClasses,
    leftHalfPanel,
    maxMintedClasses,
    mintContentClasses,
    mintTitleClasses,
    nftRendererWrapperMobileClasses,
    rightHalfPanel,
    wrapperClasses,
  } from './classes';

  let windowSize: 'sm' | 'md' | 'lg' = 'md';

  const buttonClasses = classNames('mt-6');

  $: canMint = false;
  $: totalSupply = 0;
  $: mintMax = 0;
  $: progress = Math.floor((totalSupply / 888) * 100);

  const mintState = getContext<IMint>('mint');

  $: isReady = true;

  $: totalMintCount = 0;

  $: gasCost = 0;
  $: isCalculating = false;

  $: totalMintCount, calculateGasCost();

  async function calculateGasCost() {
    try {
      isCalculating = true;

      gasCost = await Token.estimateMintGasCost();
      mintMax = await Token.maxSupply();
      totalSupply = await Token.totalSupply();

      isCalculating = false;
    } catch (e) {
      console.warn(e);
      isCalculating = false;
    }
  }

  async function load() {
    isReady = true;

    canMint = await Token.canMint();
    if (!canMint) {
      return;
    }
    totalMintCount = await User.totalWhitelistMintCount();
  }

  connectedSourceChain.subscribe(async () => {
    await load();

    const { config } = getConfig();
    const account = getAccount(config);
    if (!account || !account.address) {
      mintState.set({ ...$mintState, address: zeroAddress });
      return;
    }
    mintState.set({ ...$mintState, totalMintCount, address: account.address.toLowerCase() as IAddress });
  });

  async function mint() {
    mintState.set({
      ...$mintState,
      isModalOpen: true,

      isMinting: true,
    });

    // ensure that the input values are numbers
    totalMintCount = parseInt(totalMintCount.toString());

    mintState.set({ ...$mintState, totalMintCount: totalMintCount });
    try {
      const tokenIds = await Token.mint({
        freeMintCount: totalMintCount,
        onTransaction: (txHash: string) => {
          mintState.set({ ...$mintState, txHash });
        },
      });
      mintState.set({ ...$mintState, tokenIds });
    } catch (e) {
      console.warn(e);
      //showMintConfirmationModal = false
      mintState.set({ ...$mintState, isModalOpen: false });
    }
    mintState.set({ ...$mintState, isMinting: false });

    await load();
  }
</script>

<div class={wrapperClasses}>
  {#if isReady}
    {#if windowSize !== 'sm'}
      <div class={leftHalfPanel}>
        <NftRenderer />
      </div>
    {/if}
    <div class={rightHalfPanel}>
      <!-- svelte-ignore missing-declaration -->
      {#if windowSize === 'sm'}
        <div class={nftRendererWrapperMobileClasses}>
          <NftRenderer />
        </div>
      {/if}
      <div class={mintTitleClasses}>{$t('content.mint.title')}</div>

      <p class={mintContentClasses}>
        {$t('content.mint.text')}
      </p>

      <div class={infoRowClasses}>
        <div class={counterClasses}>
          <div class={currentMintedClasses}>#{totalSupply}</div>
          <div class={maxMintedClasses}>/ {mintMax}</div>
        </div>
        <ProgressBar {progress} />
      </div>

      <div class={counterClasses}>
        <div class={eligibilityLabelClasses}>{$t('content.mint.eligibleLabel')}</div>
        <div class={eligibilityValueClasses}>{$mintState.totalMintCount}</div>
      </div>

      <Divider />

      <div class={infoRowClasses}>
        <InfoRow label="Total mints" value={$mintState.totalMintCount.toString()} />
        <InfoRow label="Gas fee" loading={isCalculating} value={`Ξ ${gasCost}`} />
      </div>

      <Button disabled={!canMint} on:click={mint} class={buttonClasses} wide block type="primary">
        {$t('buttons.mint')}</Button>
    </div>
  {:else}
    <Spinner size="lg" />
  {/if}
</div>

<ResponsiveController bind:windowSize />
