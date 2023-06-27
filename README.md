## Polkadot and Kusama Fellowship RFCs

This repository contains a number of Requests for Comment (RFCs) detailing changes to the runtime and node behavior of these networks. These RFCs are for the discussion and design of features which have been submitted for consideration to the developer fellowships of these networks, as well as targets for the fellowships' on-chain bodies to signal approval or disapproval of.

RFCs can refer to either runtime logic (including pallets, inherents, runtime APIs) or network protocols. For any RFC concerning runtime logic, the fellowship's capabilities are bounded by relay-chain governance, which is the ultimate decider of what code is adopted within block processing for the networks. As such, these RFCs are only loosely binding - the chains' governance has no obligation to accept the features as implemented. When it comes to node-side network protocols, the fellowship's vote is more strongly binding, as the governance systems of the chains can't influence the environment the block processing logic runs within.

Merged RFCs are only an indication of support for a specific design, not a commitment to an implementation of a feature on any particular timeframe or roadmap ordering.

## Opening an RFC

Anybody is allowed to open an RFC. To do so, follow these steps:
  * Copy the `0000-template.md` file into the `text` folder and rename to match the title of the RFC
  * Fill out the RFC template and open a PR.
  * Rename the file to correspond to the GitHub pull request number and update the "RFC PR" field in the file.
