==========================
Creating a Trading Company
==========================

The simulator allows the creation of shipping companies to compete in a maritime tramp trade shipping scenario.

Baseclass and functions
=======================

Any trading company has to be a subclass of TradingCompany (:py:class:`mable.cargo_bidding.TradingCompany`).
This class defines three functions which are important for the interaction with the market.

- *pre_inform* (:py:func:`mable.cargo_bidding.TradingCompany.pre_inform`) informs the company of future trades.
- *inform* (:py:func:`mable.cargo_bidding.TradingCompany.inform`) informs the company of the trades in the current auction and expects bids.
- *receive* (:py:func:`mable.cargo_bidding.TradingCompany.pre_inform`) informs the company of the trade contracts that have been won and expects the company to schedule them.

Default Implementation
======================

By default TradingCompany (:py:class:`mable.cargo_bidding.TradingCompany`) does not do anything in pre_inform and
schedules trades for bidding and inclusion into the actual schedules in the same way.
The default implementation also always bids zero.
Specifically, the default implementation is as follows.

.. code-block:: python

    class TradingCompany:

        def pre_inform(self, trades, time):
            pass

        def inform(self, trades, *args, **kwargs):
            proposed_scheduling = self.propose_schedules(trades)
            scheduled_trades = proposed_scheduling.scheduled_trades
            trades_and_costs = [
                (x, proposed_scheduling.costs[x]) if x in proposed_scheduling.costs
                else (x, 0)
                for x in scheduled_trades]
            bids = [Bid(amount=cost, trade=one_trade) for one_trade, cost in trades_and_costs]
            return bids

        def receive(self, contracts, auction_ledger=None, *args, **kwargs):
            trades = [one_contract.trade for one_contract in contracts]
            scheduling_proposal = self.propose_schedules(trades)
            rejected_trades = self.apply_schedules(scheduling_proposal.schedules)
            if len(rejected_trades) > 0:
                logger.error(f"{len(rejected_trades)} rejected trades.")

        def propose_schedules(self, trades):
            schedules = {}
            scheduled_trades = []
            i = 0
            while i < len(trades):
                current_trade = trades[i]
                is_assigned = False
                j = 0
                while j < len(self._fleet) and not is_assigned:
                    current_vessel = self._fleet[j]
                    current_vessel_schedule = schedules.get(current_vessel, current_vessel.schedule)
                    new_schedule = current_vessel_schedule.copy()
                    new_schedule.add_transportation(current_trade)
                    if new_schedule.verify_schedule():
                        schedules[current_vessel] = new_schedule
                        scheduled_trades.append(current_trade)
                        is_assigned = True
                    j += 1
                i += 1
            return ScheduleProposal(schedules, scheduled_trades, {})
