---
layout: post
category: blockchain
---

If you distribute rewards proportionally by iterating through a user list, that's O(N). Yes, it's simple, but fundamentally broken for blockchains. As the number of users grows, gas costs go through the roof. This makes scaling impossible and destroys real-world viability for any protocol. Blockchain is not a playground where inefficiency is forgiven. If your design can't handle thousands or millions of users without blowing up fees, it's dead on arrival. In this post, I break down three mathematical approaches that solve this problem at the root, letting you achieve true O(1) pull-based reward distribution no matter how big your user base gets.

## Table of Contents
- [Fixed Stake Reward Distribution](#fixed-stake-reward-distribution)
- [Variable Stake Reward Distribution](#variable-stake-reward-distribution)

## [Fixed Stake Reward Distribution](#fixed-stake-reward-distribution)
In 2018, Bogdan et al.[^1] proposed a scalable on-chain reward distribution method in their paper "Scalable Reward Distribution on the Ethereum Blockchain." The core idea is to globally track the cumulative reward per unit stake, denoted as $S$, updating at each interval, where an interval is any event that changes the total deposit or distributes a reward.

This is summarized by:

$$
S = \sum_{t=0} \frac{reward_{t}}{T_{t}}
$$

where $T_t$ is the total deposit at interval $t$ and $j$ denotes participant. If participant $j$ deposits $stake_{j,t}$ at a time $t$, and withdraws at $t_2 > t_1$, the total reward for $j$ at $t_2$ can be computed using the global array $S_t$.

$$
reward_{j,t} = stake_{j,t} \times (S_{t} - S_{t-1})
$$

More memory-effeiciently, instead of storing all $S_{j,t}$ values globally, we can use a map $S_{j,0}$ to record the value of $S$ at the time of participant $j$'s initial deposit. When participant $j$ withdraws their entire stake, the reward is calculated as:

$$
reward_{j,t} = stake_{j,t} \times (S - S_{deposit})
$$

This is the proof of concept in Python, not for production use. You can also find it in this [gist](https://gist.github.com/suneastrex27/e25adb73d17eb82755ef5c20c8b35a29).

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
This is a proof of concept for the Bogdan reward distribution algorithm.
It is a simple implementation that demonstrates the basic functionality
of the algorithm. It is not intended for production use.
"""


class Bogdan:
    def __init__(self):
        self.T = 0.0  # Total amount of deposit
        self.S = 0.0  # (rewards distributed / total deposit) per a stake
        # Dictionary to hold staker information: {staker: (amount, current_S)}
        self.stakers = {}

    def deposit(self, staker, amount):
        """Deposit an amount to the total."""
        if amount <= 0:
            raise ValueError("Deposit amount must be positive.")
        self.T += amount

        self.stakers.setdefault(staker, (0, self.S))
        self.stakers[staker] = (amount, self.S)  # Store amount and current S
        print(f"Deposited {amount} from {staker}. Total deposit: {self.T}")

    def distribute_rewards(self, rewards):
        """Distribute rewards based on the total deposit."""
        if rewards <= 0:
            raise ValueError("Rewards must be positive.")
        if self.T == 0:
            raise ValueError("Total deposit is zero, cannot distribute rewards.")

        self.S += rewards / self.T
        print(f"Distributed {rewards} rewards. New S: {self.S}")

    def withdraw(self, staker):
        """Withdraw the staker's deposit and rewards."""
        if staker not in self.stakers:
            raise ValueError(f"No deposits found for staker: {staker}")

        amount, current_S = self.stakers[staker]
        if amount <= 0 or self.S <= current_S:
            raise ValueError("Insufficient funds or rewards for withdrawal.")

        reward = (self.S - current_S) * amount
        if reward < 0:
            raise ValueError("No rewards available for withdrawal.")

        del self.stakers[staker]
        self.T -= amount
        print(
            f"{staker} withdrew {amount} and received {reward} rewards. Total deposit now: {self.T}"
        )

    def get_total_deposit(self):
        """Get the total amount of deposit."""
        return self.T

    def get_staker_info(self, staker):
        """Get the information of a specific staker."""
        if staker not in self.stakers:
            raise ValueError(f"No deposits found for staker: {staker}")

        amount, current_S = self.stakers[staker]
        return {
            "amount": amount,
            "current_S": current_S,
            "total_rewards": amount * (self.S - current_S),
        }


# Example usage
if __name__ == "__main__":
    bogdan = Bogdan()
    bogdan.deposit("Alice", 100)
    bogdan.deposit("Bob", 50)
    bogdan.distribute_rewards(10)
    bogdan.distribute_rewards(10)

    print(bogdan.get_staker_info("Alice"))
    print(bogdan.get_staker_info("Bob"))

    bogdan.withdraw("Alice")
    bogdan.withdraw("Bob")

    print(f"Total deposit after withdrawals: {bogdan.get_total_deposit()}")
```

## [Variable Stake Reward Distribution](#variable-stake-reward-distribution)
In 2019, Onur Solmaz proposed a new method in his blog[^2]. This is an optimized upgrade of the Bogdan method, as it enables proportional reward distribution even when stake sizes change. In practice, participants rarely adjust their stake size, but the need for variable stake support still exists.

Starting from Bogdan's formula:

$$
S = \sum_{t=0} \frac{reward_{t}}{T_{t}}
$$

The total reward $N_j$ that participant $j$ receives is then calculated as:

$$
N_{j} = \sum_{t=0} stake_{j,t} \times \frac{reward_{t}}{T_{t}}
$$

Solmaz applied Abel's summation by parts to transform this double sequence sum into a form that captures the overall change, not just the individual terms. In mathematics, summation by parts rewrites the sum of products of two sequences into alternative forms. This is also known as Abel’s lemma or the Abel transformation.

Suppose $a_n = (a_0, a_1, ..., a_n)$ and $b_n = (b_0, b_1, ..., b_n)$ are two sequences. Then,

**Identity**:

$$
\sum_{i=0}^{n} a_{i}b_{i} = a_n \sum_{j=0}^{n} b_j - \sum_{i=1}^{n}((a_i - a_{i-1}) \sum_{j=0}^{i-1}b_j)
$$

The principle is simple. Instead of summing every individual product, you break the sum into two parts, a boundary term and a sum of differences. By focusing only on incremental changes and the final boundary, complex summations become much more tractable.

We assume $n+1$ rewards, indexed by $t=0, ..., n$, and apply the identity to total rewards to obtain:

$$
N_{j} = stake_{j,n} \sum_{t=0}^{n} \frac{reward_t}{T_t} - \sum_{t=1}^n ((stake_{j,t} - stake_{j,t-1}) \sum_{t=0}^{t-1} \frac{reward_t}{T_t})
$$

We define $R$, short for reward per token, as follows:

$$
R_t = \sum_{k=0}^{t} \frac{reward_k}{T_k}
$$

The change in stake is given by the forward difference operator $\Delta$ between rewards $t-1$ and $t$:

$$
\Delta stake_{j,t} = stake_{j,t} - stake_{j,t-1}
$$

Thus, we can write:

$$
N_j = stake_{j,n} * R_n - \sum_{t=1}^{n} (\Delta stake_{j,t} \times R_{t-1})
$$

While Bogdan's approach tracks $S$ (reward per token) globally, this approach instead keeps track of:

$$
tally_{j,n} = \sum_{t=1}^{n} (\Delta stake_{j,t} \times R_{t-1})
$$

$\Delta stake_{j,t}$ is positive for deposits and negative for withdrawals. Finally, we can calculate $N_j$ as:

$$
N_j = stake_{j,n} * R_n - tally_{j,n}
$$

This is the proof of concept in Python, not for production use. You can also find it in this [gist](https://gist.github.com/suneastrex27/a08d8f51ead40fb1d3060fe07c757dd4).

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
This is a proof of concept for the Solmaz reward distribution algorithm.
It is a simple implementation that demonstrates the basic functionality
of the algorithm. It is not intended for production use.

The Solmaz algorithm is designed to handle deposits and reward distributions
in a straightforward manner, similar to the Bogdan algorithm but with changeable stake sizes.
"""


class Solmaz:
    def __init__(self):
        self.total_stake = 0
        self.reward_per_token = 0
        self.stake = {}
        self.reward_tally = {}

    def deposit_stake(self, staker, amount):
        if amount <= 0:
            raise ValueError("Deposit amount must be positive.")
        if staker not in self.stake:
            self.stake[staker] = 0
            self.reward_tally[staker] = 0
        self.stake[staker] += amount
        self.reward_tally[staker] += self.reward_per_token * amount
        self.total_stake += amount

    def distribute_rewards(self, rewards):
        if rewards <= 0:
            raise ValueError("Rewards must be positive.")
        if self.total_stake == 0:
            raise ValueError("Total stake is zero, cannot distribute rewards.")
        self.reward_per_token += rewards / self.total_stake

    def calculate_rewards(self, staker):
        if staker not in self.stake:
            return 0
        return self.stake[staker] * self.reward_per_token - self.reward_tally[staker]

    def withdraw_stake(self, staker, amount):
        if staker not in self.stake:
            raise ValueError("No stake found for staker: %s" % staker)
        if amount <= 0 or amount > self.stake[staker]:
            raise ValueError("Invalid withdrawal amount.")
        self.stake[staker] -= amount
        self.reward_tally[staker] -= self.reward_per_token * amount
        self.total_stake -= amount
        if self.stake[staker] == 0:
            del self.stake[staker]
            del self.reward_tally[staker]

    def withdraw_all(self, staker):
        if staker not in self.stake:
            raise ValueError("No stake found for staker: %s" % staker)
        rewards = self.calculate_rewards(staker)
        stake_amt = self.stake[staker]
        self.withdraw_stake(staker, stake_amt)
        return stake_amt, rewards


# Example usage
if __name__ == "__main__":
    solmaz = Solmaz()
    solmaz.deposit_stake("Alice", 100)
    solmaz.deposit_stake("Bob", 100)
    solmaz.distribute_rewards(10)
    solmaz.deposit_stake("Alice", 100)
    solmaz.distribute_rewards(10)

    print("Alice rewards:", solmaz.calculate_rewards("Alice"))
    print("Bob rewards:", solmaz.calculate_rewards("Bob"))

    a_stake, a_reward = solmaz.withdraw_all("Alice")
    b_stake, b_reward = solmaz.withdraw_all("Bob")
    print(f"Alice withdrew {a_stake} stake and {a_reward} rewards.")
    print(f"Bob withdrew {b_stake} stake and {b_reward} rewards.")
    print("Total stake after withdrawals:", solmaz.total_stake)
```

## Reference
[^1]: Batog, B., Nowostawski, M., Sylwestrzak, W., "Scalable Reward Distribution on the Ethereum Blockchain," 2018. Available: https://batog.info/papers/scalable-reward-distribution.pdf
[^2]: Solmaz, "Scalable Reward Distribution with Changing Stake Sizes," https://solmaz.io/2019/02/24/scalable-reward-changing/