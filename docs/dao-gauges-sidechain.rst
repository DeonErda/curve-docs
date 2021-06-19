.. _dao-gauges-sidechain:

========================================
The Curve DAO: Gauges for EVM Sidechains
========================================

In addition to Ethereum, Curve is active on several `sidechains <https://docs.ethhub.io/ethereum-roadmap/layer-2-scaling/sidechains/>`_.

The Curve DAO is sufficiently complex that it cannot be easily bridged outside of Ethereum, however aspects of functionality (including CRV emissions) are capable on the various sidechains where pools are active.

Source code for the smart contracts used in sidechain emissions are available on `Github <https://github.com/curvefi/curve-dao-contracts/tree/master/contracts/gauges/sidechain>`_.

.. note::

    Each sidechain comes with it's own set of tradeoffs between security, scalability and cost of use. The technical specifications and security considerations of each sidechain is outside the scope of this documentation, however we encourage all users to do their own research prior to transferring funds off of Ethereum and onto a sidechain.

Implementation Details
======================

At a high level, the process of CRV distribution on sidechain gauges is as follows:

1. On Ethereum, a ``RootChainGauge`` contract mints allocated CRV each week and transfers it over the bridge.

    At the beginning of each epoch week, a call is made to the :func:`checkpoint<RootChainGauge.checkpoint>` function within each gauge. This function mints all of the allocated CRV for the previous week, and transfers them over the bridge to another contract deployed at the same address on the related sidechain. Emissions are delayed by one week in order to avoid exceeding the max allowable supply of CRV.

    Checkpointing may be performed by anyone. However, for chains that use the `AnySwap bridge <https://anyswap.exchange/bridge>`_ the checkpoint must happen via the ``CheckpointProxy`` contract.

2. On the sidechain, CRV is received into a ``ChildChainStreamer`` contract and then streamed out to a ``RewardsOnlyGauge``.

    The bridge automatically transfers CRV into a streamer contract, deployed at the same address on the sidechain as the gauge is on Ethereum. Once the CRV has arrived, a call is made to :func:`notify_reward_amount<ChildChainStreamer.notify_reward_amount>`. This call updates the local accounting and streams the balance out linearly over the next seven days.

3. Liquidity providers who have staked their LP tokens in the ``RewardsOnlyGauge`` may claim their CRV.

    The sidechain gauge is a simplified version of the gauges used on Ethereum. It handles CRV as though it were any other 3rd-party reward token, evenly distributing between stakers based on the deposited balances as the time the token is received.

Sidechain Gauge Types
=====================

Each sidechain where Curve operates is assigned a unique gauge type. By grouping gauges in this way, the Curve DAO may choose to end emissions for a specific sidechain by voting a type weight of zero to that sidechain's gauge type.

The gauge types are as follows:

   * Fantom: ``1``
   * Polygon (Matic): ``2``

RootChainGauge
==============

``RootChainGauge`` is a simplified liquidity gauge contract used for bridging CRV from Ethereum to a sidechain. Each root gauge is added to the gauge controller and receives gauge weight votes to determine emissions for a sidechain pool.

The gauge cannot be directly staked into. There is one important external function:

.. py:function:: RootChainGauge.checkpoint(): nonpayable

    Mints all allocated CRV emissions for the gauge, and transfers across the bridge.

    This function should be called once per week, immediately after the start of the epoch week. Subsequent calls within the same epoch week have no effect.

    For gauges that use the AnySwap bridge, this function is guarded and can only be called indirectly via ``CheckpointProxy.checkpoint_many``.

ChildChainStreamer
==================

``ChildChainStreamer`` is a simple reward streaming contract. The logic is similar to that of the Synthetix `staking rewards contract <https://github.com/Synthetixio/synthetix/blob/master/contracts/StakingRewards.sol>`_.

For each ``RootChainGauge`` deployed on Ethereum, a ``ChildChainStreamer`` is deployed at the same address on the related sidechain. CRV tokens that are sent over the bridge are transferred into the streamer. From there they are released linearly over seven days, to the gauge where LPs ultimately stake and claim them.

.. py:function:: ChildChainStreamer.notify_reward_amount(token: address):

    Notify the contract of a newly received reward. This updates the local accounting and streams the reward over a preset period (typically seven days).

    If the previous reward period has already expired, this function is callable by anyone. When there is an active reward period it may only be called by the designated reward distributor account. Without this check, it would be possible to exploit the system by repeatedly calling to extend an active reward period and thus dragging out the duration over which the rewards are released.

    Reverts if ``token`` is not registered as a reward within the contract, or if no extra balance of ``token`` was added prior to the call.

RewardsOnlyGauge
================

``RewardsOnlyGauge`` is a simplified version of the same gauge contract used on Ethereum. The logic around CRV emissions and minting has been removed - it only deals with distribution of externally received rewards.

The API for this contract is similar to that of ``LiquidityGaugeV3``.

RewardClaimer
=============

``RewardClaimer`` is a minimal passthrough contract that allows claiming from multiple reward streamers. For example the am3CRV pool on Polygon utilizes this contract to receive both CRV emissions bridged across from Ethereum, as well as WMATIC rewards supplied via a ``RewardStreamer`` contract. The ``RewardsOnlyGauge`` calls the ``RewardClaimer`` as a way to retrieve both the CRV and WMATIC rewards.
