### Revolutionizing Design-Bid-Build (DBB) Construction with Blockchain and zkSync: A Technical Overview


![11-Tips-for-Effective-Construction-Site-Management-Procore-Blog-Hero-](https://github.com/user-attachments/assets/20fd0196-eddc-4a5e-89c2-adff661321b0)


The construction industry is ripe for disruption. The traditional Design-Bid-Build (DBB) has been the source of construction Industry. Traditional Design-Bid-Build (DBB) methods often suffer from inefficiencies, disputes, and lack of transparency. Leveraging blockchain technology and Layer 2 scaling solutions like zkSync era can transform the way DBB projects are managed, making the process more secure, efficient, and transparent.

In this technical article, we will not only explore the advantages of using blockchain and zkSync in DBB projects but also dive into the specifics of a smart contract implementation that automates the DBB process, ensuring smoother execution and minimizing disputes.

#### The Challenges of Traditional DBB Construction

The Design-Bid-Build process typically involves three stages:
1. **Design**: The project owner hires an architect or designer to create a blueprint for the project.
2. **Bidding**: Contractors submit bids based on the design specifications, and the owner selects the most favorable bid.
3. **Build**: The selected contractor executes the project according to the design and specifications, receiving payments in stages as work progresses.

While effective, this process has several pain points:
- **Delayed Payments**: Contractors often face delays in receiving payments due to manual processes and disputes over milestone completion.
- **Lack of Transparency**: The bidding process can be opaque, leading to mistrust between stakeholders.
- **Dispute Resolution**: Disputes over contract terms and performance often escalate into costly legal battles.

Blockchain technology addresses these challenges by introducing decentralization, transparency, and automation through smart contracts.

#### Leveraging Blockchain in DBB Projects

Blockchain provides a decentralized and transparent platform where all transactions are recorded immutably. Smart contracts enable the automation of payments, dispute resolution, and other contractual obligations. Here's how blockchain improves the DBB process:
- **Transparency**: All bids, payments, and disputes are recorded on the blockchain, ensuring that no party can alter or tamper with the data.
- **Automation**: Smart contracts automatically execute payments when predefined conditions are met, eliminating the need for manual processing.
- **Security**: Blockchain's decentralized nature ensures that no single entity can manipulate the system, reducing the risk of fraud.

#### zkSync: Solving Scalability and Cost Issues

One challenge with using blockchain for DBB projects is the cost and speed of transactions. On Ethereum’s Layer 1, transaction fees can be high, and network congestion can lead to delays. This is where zkSync, a Layer 2 scaling solution, comes into play.

zkSync leverages zero-knowledge rollups to batch transactions off-chain and then submit a proof to the Ethereum mainnet, reducing transaction fees and increasing throughput while maintaining security. By using zkSync, DBB projects can enjoy:
- **Lower Fees**: Transaction costs are significantly reduced, making smart contract execution more affordable.
- **Faster Confirmations**: Transactions are confirmed quickly, even during high network traffic.
- **Security**: zkSync inherits Ethereum’s security guarantees, ensuring that transactions are safe from tampering.

Now, let’s take a look at how the DesignByBid (DBB) smart contract integrates blockchain and zkSync to automate the DBB process.

---

### The DesignByBid Smart Contract: A Technical Breakdown

The DesignByBid smart contract automates the DBB process, ensuring that every step, from project posting to bidding and milestone payments, is handled securely and transparently. Below is the complete smart contract that will be using in this article. You can check this ([repo](https://github.com/embolaweb3/design-by-bid)) for complete source code. 


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract DesignByBid {
    struct Project {
        uint id;
        address owner;
        string description;
        uint budget;
        uint deadline;
        bool active;
        address selectedBidder;
        uint[] milestones;
        bool[] milestonePaid;
        bool disputeRaised;
        uint selectedBidIndex;
    }

    struct Bid {
        uint projectId;
        address bidder;
        uint bidAmount;
        uint completionTime;
        uint[] proposedMilestones;
        bool selected;
    }

    struct Dispute {
        uint projectId;
        address disputant;
        string reason;
        mapping(address => bool) votes;
        uint yesVotes;
        uint noVotes;
        bool resolved;
    }

    uint public projectCount;
    uint public bidCount;
    uint public disputeCount;

    mapping(uint => Project) public projects;
    mapping(uint => Bid[]) public bids;
    mapping(uint => Dispute) public disputes;

    event ProjectPosted(uint projectId, address owner, string description, uint budget, uint deadline);
    event BidSubmitted(uint bidId, uint projectId, address bidder, uint bidAmount, uint completionTime);
    event BidSelected(uint projectId, address selectedBidder);
    event MilestonePaid(uint projectId, uint milestoneIndex);
    event DisputeRaised(uint disputeId, uint projectId, address disputant, string reason);
    event DisputeResolved(uint disputeId, bool result);

    modifier onlyProjectOwner(uint _projectId) {
        require(msg.sender == projects[_projectId].owner, "Only the project owner can perform this action");
        _;
    }

    modifier onlySelectedBidder(uint _projectId) {
        require(msg.sender == projects[_projectId].selectedBidder, "Only the selected bidder can perform this action");
        _;
    }

    // Project owner posts a new project
    function postProject(string memory _description, uint _budget, uint _deadline, uint[] memory _milestones) public {
        require(_milestones.length > 0, "There must be at least one milestone");
        projectCount++;
        projects[projectCount] = Project(projectCount, msg.sender, _description, _budget, _deadline, true, address(0), _milestones, new bool[](_milestones.length), false, 0);
        emit ProjectPosted(projectCount, msg.sender, _description, _budget, _deadline);
    }

    // Contractors submit bids
    function submitBid(uint _projectId, uint _bidAmount, uint _completionTime, uint[] memory _proposedMilestones) public {
        require(projects[_projectId].active, "Project is not active");
        require(_proposedMilestones.length == projects[_projectId].milestones.length, "Milestones count mismatch");

        bidCount++;
        bids[_projectId].push(Bid(_projectId, msg.sender, _bidAmount, _completionTime, _proposedMilestones, false));
        emit BidSubmitted(bidCount, _projectId, msg.sender, _bidAmount, _completionTime);
    }

    // Project owner selects a bid
    function selectBid(uint _projectId, uint _bidIndex) public onlyProjectOwner(_projectId) {
        require(bids[_projectId][_bidIndex].bidder != address(0), "Bid does not exist");
        require(!projects[_projectId].disputeRaised, "Cannot select bid during dispute");

        projects[_projectId].selectedBidder = bids[_projectId][_bidIndex].bidder;
        bids[_projectId][_bidIndex].selected = true;
        projects[_projectId].active = false; // Project is no longer active once a bid is selected
        projects[_projectId].selectedBidIndex = _bidIndex;
        emit BidSelected(_projectId, bids[_projectId][_bidIndex].bidder);
    }

    // Release payment for a milestone
    function releaseMilestonePayment(uint _projectId, uint _milestoneIndex) public onlyProjectOwner(_projectId) {
        require(projects[_projectId].selectedBidder != address(0), "No bid selected");
        require(!projects[_projectId].milestonePaid[_milestoneIndex], "Milestone already paid");
        require(_milestoneIndex < projects[_projectId].milestones.length, "Invalid milestone index");
        require(!projects[_projectId].disputeRaised, "Cannot release payment during dispute");

        uint payment = projects[_projectId].milestones[_milestoneIndex];
        projects[_projectId].milestonePaid[_milestoneIndex] = true;
        (bool success,) = payable(projects[_projectId].selectedBidder).call{value : payment}("");
        require(success, 'Transfer failed');
        emit MilestonePaid(_projectId, _milestoneIndex);
    }

    // Raise a dispute
    function raiseDispute(uint _projectId, string memory _reason) public onlySelectedBidder(_projectId) {
        require(!projects[_projectId].disputeRaised, "Dispute already raised for this project");

        disputeCount++;
        Dispute storage dispute = disputes[disputeCount];
        dispute.projectId = _projectId;
        dispute.disputant = msg.sender;
        dispute.reason = _reason;
        projects[_projectId].disputeRaised = true;
        emit DisputeRaised(disputeCount, _projectId, msg.sender, _reason);
    }

    // Vote on a dispute (by any stakeholder)
    function voteOnDispute(uint _disputeId, bool _vote) public {
        require(!disputes[_disputeId].resolved, "Dispute already resolved");
        require(!disputes[_disputeId].votes[msg.sender], "You have already voted on this dispute");

        disputes[_disputeId].votes[msg.sender] = true;
        if (_vote) {
            disputes[_disputeId].yesVotes++;
        } else {
            disputes[_disputeId].noVotes++;
        }

        // Resolve the dispute if a majority is reached
        if (disputes[_disputeId].yesVotes > disputes[_disputeId].noVotes) {
            disputes[_disputeId].resolved = true;
            _resolveDispute(_disputeId, true);
        } else if (disputes[_disputeId].noVotes > disputes[_disputeId].yesVotes) {
            disputes[_disputeId].resolved = true;
            _resolveDispute(_disputeId, false);
        }
    }

    function _resolveDispute(uint _disputeId, bool result) internal {
        uint projectId = disputes[_disputeId].projectId;
        projects[projectId].disputeRaised = false;
        emit DisputeResolved(_disputeId, result);
    }

    // Fallback function to accept Ether
    receive() external payable {}
}

```

Below is a detailed explanation of the key components of the smart contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract DesignByBid {
    struct Project {
        uint id;
        address owner;
        string description;
        uint budget;
        uint deadline;
        bool active;
        address selectedBidder;
        uint[] milestones;
        bool[] milestonePaid;
        bool disputeRaised;
        uint selectedBidIndex;
    }

    struct Bid {
        uint projectId;
        address bidder;
        uint bidAmount;
        uint completionTime;
        uint[] proposedMilestones;
        bool selected;
    }

    struct Dispute {
        uint projectId;
        address disputant;
        string reason;
        mapping(address => bool) votes;
        uint yesVotes;
        uint noVotes;
        bool resolved;
    }

    uint public projectCount;
    uint public bidCount;
    uint public disputeCount;

    mapping(uint => Project) public projects;
    mapping(uint => Bid[]) public bids;
    mapping(uint => Dispute) public disputes;

    event ProjectPosted(uint projectId, address owner, string description, uint budget, uint deadline);
    event BidSubmitted(uint bidId, uint projectId, address bidder, uint bidAmount, uint completionTime);
    event BidSelected(uint projectId, address selectedBidder);
    event MilestonePaid(uint projectId, uint milestoneIndex);
    event DisputeRaised(uint disputeId, uint projectId, address disputant, string reason);
    event DisputeResolved(uint disputeId, bool result);
```

The `DesignByBid` contract facilitates a decentralized Design-Bid-Build (DBB) process. It manages **projects**, **bids**, and **disputes**, ensuring transparency and automation in construction projects.

#### Key Structs:

1. **`Project`**: Represents a construction project with:
   - `id`: Unique project ID.
   - `owner`: Address of the project owner.
   - `description`, `budget`, `deadline`: Basic project details.
   - `milestones`: Payment stages.
   - `selectedBidder`: Address of the chosen contractor.
   - `milestonePaid`: Track milestone payments.
   - `disputeRaised`: Flags if a dispute is active.
   - `active`: Indicates if the project is open for bidding.

2. **`Bid`**: Represents a contractor's bid, including:
   - `bidder`: Contractor's address.
   - `bidAmount`: Amount requested to complete the project.
   - `completionTime`: Estimated time to finish.
   - `proposedMilestones`: Suggested payment breakdown.

3. **`Dispute`**: Manages disputes through voting:
   - `disputant`: Address raising the dispute.
   - `votes`: Vote tally (yes/no).
   - `resolved`: Marks if the dispute is resolved.

#### Key Features:

- **Project Posting**: Owners create projects with milestones.
- **Bid Submission**: Contractors submit bids for projects.
- **Bid Selection**: Owners choose the winning bid, deactivating further bids.
- **Milestone Payments**: Owners release payments upon milestone completion.
- **Dispute Resolution**: Disputes are resolved via on-chain voting.

#### Events:

- **ProjectPosted, BidSubmitted, BidSelected, MilestonePaid, DisputeRaised, DisputeResolved**: These events log significant actions for transparency and off-chain tracking.


### Key Features of the Smart Contract

1. **Project Posting**
    - The `postProject` function allows the project owner to post a new construction project, defining the project description, budget, deadline, and payment milestones.
    - A `Project` struct stores the project's details, including whether the project is active, the selected bidder, and the status of each milestone.

```solidity
    function postProject(string memory _description, uint _budget, uint _deadline, uint[] memory _milestones) public {
        require(_milestones.length > 0, "There must be at least one milestone");
        projectCount++;
        projects[projectCount] = Project(projectCount, msg.sender, _description, _budget, _deadline, true, address(0), _milestones, new bool[](_milestones.length), false, 0);
        emit ProjectPosted(projectCount, msg.sender, _description, _budget, _deadline);
    }
```
   - **Milestone Payments:** The owner sets milestones that must be completed before payments are released. Milestones ensure accountability and break down large projects into manageable segments.

2. **Bidding Process**
    - Contractors can submit bids using the `submitBid` function. Bids include the bid amount, completion time, and proposed milestones, which must align with the project owner’s milestones.

```solidity
    function submitBid(uint _projectId, uint _bidAmount, uint _completionTime, uint[] memory _proposedMilestones) public {
        require(projects[_projectId].active, "Project is not active");
        require(_proposedMilestones.length == projects[_projectId].milestones.length, "Milestones count mismatch");

        bidCount++;
        bids[_projectId].push(Bid(_projectId, msg.sender, _bidAmount, _completionTime, _proposedMilestones, false));
        emit BidSubmitted(bidCount, _projectId, msg.sender, _bidAmount, _completionTime);
    }
```
   - **Transparency:** All bids are stored on-chain, ensuring transparency. The project owner can review all bids and select the most favorable one, enhancing trust between parties.

3. **Bid Selection**
    - The project owner selects the most suitable bid using the `selectBid` function. Once selected, the project is marked as inactive to prevent further bids, and the selected contractor’s address is stored in the contract.

```solidity
    function selectBid(uint _projectId, uint _bidIndex) public onlyProjectOwner(_projectId) {
        require(bids[_projectId][_bidIndex].bidder != address(0), "Bid does not exist");
        require(!projects[_projectId].disputeRaised, "Cannot select bid during dispute");

        projects[_projectId].selectedBidder = bids[_projectId][_bidIndex].bidder;
        bids[_projectId][_bidIndex].selected = true;
        projects[_projectId].active = false; // Project is no longer active once a bid is selected
        projects[_projectId].selectedBidIndex = _bidIndex;
        emit BidSelected(_projectId, bids[_projectId][_bidIndex].bidder);
    }
```
   - **Immutable Record:** The bid selection process is recorded on the blockchain, making it immutable and verifiable.

4. **Milestone Payments**
    - As work progresses, the project owner can release payments for completed milestones. The `releaseMilestonePayment` function transfers the payment directly to the contractor’s address, ensuring that funds are securely handled through the smart contract.

```solidity
    function releaseMilestonePayment(uint _projectId, uint _milestoneIndex) public onlyProjectOwner(_projectId) {
        require(projects[_projectId].selectedBidder != address(0), "No bid selected");
        require(!projects[_projectId].milestonePaid[_milestoneIndex], "Milestone already paid");
        require(_milestoneIndex < projects[_projectId].milestones.length, "Invalid milestone index");
        require(!projects[_projectId].disputeRaised, "Cannot release payment during dispute");

        uint payment = projects[_projectId].milestones[_milestoneIndex];
        projects[_projectId].milestonePaid[_milestoneIndex] = true;
        (bool success,) = payable(projects[_projectId].selectedBidder).call{value : payment}("");
        require(success, 'Transfer failed');
        emit MilestonePaid(_projectId, _milestoneIndex);
    }
```
   - **Automation:** Payments are automatically handled by the smart contract, ensuring that contractors are paid on time and minimizing human errors.

5. **Dispute Resolution**
    - If disputes arise, the contractor can raise a dispute using the `raiseDispute` function. Disputes are resolved through decentralized voting, where stakeholders vote on the outcome.

```solidity
    function raiseDispute(uint _projectId,string memory _reason) public {
        require(projects[_projectId].active, "Project is not active");
        require(projects[_projectId].selectedBidder == msg.sender || msg.sender == projects[_projectId].owner, "Only involved parties can raise disputes");

        disputeCount++;
        disputes[disputeCount] = Dispute(_projectId, msg.sender, _reason, 0, 0, false);
        projects[_projectId].disputeRaised = true;
        emit DisputeRaised(disputeCount, _projectId, msg.sender, _reason);
    }
    
    // Voting and resolving the dispute can be implemented here
```
   - **Decentralized Justice:** Disputes are resolved on-chain, with voting mechanisms ensuring fairness and reducing the need for expensive legal proceedings.

---

### Deploying on ZkSync Era Sepolia Testnet

The DesignByBid smart contract can be deployed on zkSync to benefit from lower fees and faster transactions. Here’s an outline of how you can deploy your smart contract on zkSync:


1. **Compile the Smart Contract**: Ensure that your contract is compiled with no error. For easy compilation and deployment, you can check zksync documentations for various environments
    such as [Hardhat](https://docs.zksync.io/build/tooling/hardhat/getting-started) or [Foundry](https://docs.zksync.io/build/tooling/foundry/overview). zkSync uses Solidity, so no significant changes are necessary for basic contracts.

3. **Deploy on zkSync Testnet**: Deploy your contract on the zkSync sepolia testnet to verify that everything works as expected. Depending on the environment you are using it.


#### Benefits of zkSync Deployment
- **Cost-Effective Execution**: zkSync reduces gas fees, making your smart contracts more affordable for users.
- **Speed**: Faster transaction finality ensures that project milestones and payments are processed quickly.
- **Scalability**: zkSync can handle a large number of transactions without congestion, ensuring that even large-scale construction projects can operate smoothly.

---

### Conclusion

By integrating blockchain technology with zkSync, the DesignByBid smart contract revolutionizes the traditional DBB process. It automates key aspects such as bidding, milestone payments, and dispute resolution, ensuring that the entire process is transparent, efficient, and secure.


