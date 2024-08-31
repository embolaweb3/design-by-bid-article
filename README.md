# Revolutionizing Design-Bid-Build (DBB) Construction with Blockchain and zkSync: A Technical Overview

![11-Tips-for-Effective-Construction-Site-Management-Procore-Blog-Hero-](https://github.com/user-attachments/assets/20fd0196-eddc-4a5e-89c2-adff661321b0)

## Table of Contents
1. [Introduction](#introduction)
2. [The Challenges of Traditional DBB Construction](#the-challenges-of-traditional-dbb-construction)
3. [Leveraging Blockchain in DBB Projects](#leveraging-blockchain-in-dbb-projects)
4. [zkSync: Solving Scalability and Cost Issues](#zksync-solving-scalability-and-cost-issues)
5. [The DesignByBid Smart Contract: A Technical Breakdown](#the-designbybid-smart-contract-a-technical-breakdown)
   - [Key Structs](#key-structs)
   - [Key Features](#key-features)
   - [Events](#events)
6. [Deploying on zkSync Era Sepolia Testnet](#deploying-on-zksync-era-sepolia-testnet)
7. [Frontend Implementation using Next.js, TypeScript, and TailwindCSS](#frontend-implementation-using-nextjs-typescript-and-tailwindcss)
   - [Setting Up the Next.js Project](#setting-up-the-nextjs-project)
   - [Creating the User Interface](#creating-the-user-interface)
   - [Connecting to the Smart Contract](#connecting-to-the-smart-contract)
8. [Conclusion](#conclusion)

---

## Introduction

The construction industry is ripe for disruption. The traditional Design-Bid-Build (DBB) method has been the cornerstone of construction, but it often suffers from inefficiencies, disputes, and lack of transparency. Leveraging blockchain technology and Layer 2 scaling solutions like zkSync era can transform how DBB projects are managed, making the process more secure, efficient, and transparent.

In this technical article, we will explore the advantages of using blockchain and zkSync in DBB projects and dive into the specifics of a smart contract implementation that automates the DBB process, ensuring smoother execution and minimizing disputes.

## The Challenges of Traditional DBB Construction

The Design-Bid-Build process typically involves three stages:
1. **Design**: The project owner hires an architect or designer to create a blueprint for the project.
2. **Bidding**: Contractors submit bids based on the design specifications, and the owner selects the most favorable bid.
3. **Build**: The selected contractor executes the project according to the design and specifications, receiving payments in stages as work progresses.

While effective, this process has several pain points:
- **Delayed Payments**: Contractors often face delays in receiving payments due to manual processes and disputes over milestone completion.
- **Lack of Transparency**: The bidding process can be opaque, leading to mistrust between stakeholders.
- **Dispute Resolution**: Disputes over contract terms and performance often escalate into costly legal battles.

Blockchain technology addresses these challenges by introducing decentralization, transparency, and automation through smart contracts.

## Leveraging Blockchain in DBB Projects

Blockchain provides a decentralized and transparent platform where all transactions are recorded immutably. Smart contracts enable the automation of payments, dispute resolution, and other contractual obligations. Here's how blockchain improves the DBB process:
- **Transparency**: All bids, payments, and disputes are recorded on the blockchain, ensuring that no party can alter or tamper with the data.
- **Automation**: Smart contracts automatically execute payments when predefined conditions are met, eliminating the need for manual processing.
- **Security**: Blockchain's decentralized nature ensures that no single entity can manipulate the system, reducing the risk of fraud.

## zkSync: Solving Scalability and Cost Issues


One challenge with using blockchain for DBB projects is the cost and speed of transactions. On Ethereum’s Layer 1, transaction fees can be high, and network congestion can lead to delays. This is where zkSync, a Layer 2 scaling solution, comes into play.

zkSync leverages zero-knowledge rollups to batch transactions off-chain and then submit a proof to the Ethereum mainnet, reducing transaction fees and increasing throughput while maintaining security. By using zkSync, DBB projects can enjoy:
- **Lower Fees**: Transaction costs are significantly reduced, making smart contract execution more affordable.
- **Faster Confirmations**: Transactions are confirmed quickly, even during high network traffic.
- **Security**: zkSync inherits Ethereum’s security guarantees, ensuring that transactions are safe from tampering.

Now, let’s take a look at how the DesignByBid (DBB) smart contract integrates blockchain and zkSync to automate the DBB process.

## The DesignByBid Smart Contract: A Technical Breakdown

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


### Key Structs

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

### Key Features

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

### Events

- **ProjectPosted, BidSubmitted, BidSelected, MilestonePaid, DisputeRaised, DisputeResolved**: These events log significant actions for transparency and off-chain tracking.

## Deploying on zkSync Era Sepolia Testnet


The DesignByBid smart contract can be deployed on zkSync to benefit from lower fees and faster transactions. Here’s an outline of how you can deploy your smart contract on zkSync:


1. **Compile the Smart Contract**: Ensure that your contract is compiled with no error. For easy compilation and deployment, you can check zksync documentations for various environments
    such as [Hardhat](https://docs.zksync.io/build/tooling/hardhat/getting-started) or [Foundry](https://docs.zksync.io/build/tooling/foundry/overview). zkSync uses Solidity, so no significant changes are necessary for basic contracts.

3. **Deploy on zkSync Testnet**: Deploy your contract on the zkSync sepolia testnet to verify that everything works as expected. Depending on the environment you are using it.


## Frontend Implementation using Next.js, TypeScript, and TailwindCSS

### Setting Up the Next.js Project

To build the frontend for the DesignByBid smart contract, we will use Next.js with TypeScript and TailwindCSS for styling.

 **Initialize the Next.js Project**:
   - Run the following commands to set up a new Next.js project with TypeScript and Tailwindcss by:
   - Ensure you select Yes to use `Tailwind CSS` and `Typescript`
     ```bash
     npx create-next-app@latest designbybid-frontend
     cd designbybid-frontend
     ```

### Creating the User Interface

The user interface will include the following components:
- **Project List**: Display a list of projects that users can view and bid on.
- **Project Details**: Show details for a selected project, including the bids submitted and the milestones set.
- **Bid Submission Form**: Allow contractors to submit bids for projects.
- **Milestone Payment**: Let project owners release payments for completed milestones.

1. **Project List Component**:
   - Create a new file `components/ProjectList.tsx`:
     ```tsx
     import React from 'react';

     const ProjectList = ({ projects }) => {
       return (
         <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
           {projects.map(project => (
             <div key={project.id} className="p-4 border rounded-lg shadow-md">
               <h3 className="text-xl font-bold">{project.description}</h3>
               <p>Budget: {project.budget}</p>
               <p>Deadline: {new Date(project.deadline).toLocaleDateString()}</p>
               <button className="mt-2 p-2 bg-blue-500 text-white rounded-lg">
                 View Details
               </button>
             </div>
           ))}
         </div>
       );
     };

     export default ProjectList;
     ```

2. **Project Details Component**:
   - Create a new file `components/ProjectDetails.tsx`:
     ```tsx
     import React from 'react';

     const ProjectDetails = ({ project }) => {
       return (
         <div className="p-4 border rounded-lg shadow-md">
           <h2 className="text-2xl font-bold">{project.description}</h2>
           <p>Budget: {project.budget}</p>
           <p>Deadline: {new Date(project.deadline).toLocaleDateString()}</p>
           <h3 className="mt-4 text-xl font-bold">Bids:</h3>
           <ul>
             {project.bids.map((bid, index) => (
               <li key={index}>
                 Bid by {bid.bidder}: {bid.bidAmount} - Completion Time: {bid.completionTime}
               </li>
             ))}
           </ul>
           <h3 className="mt-4 text-xl font-bold">Milestones:</h3>
           <ul>
             {project.milestones.map((milestone, index) => (
               <li key={index}>
                 Milestone {index + 1}: {milestone} - Paid: {project.milestonePaid[index] ? "Yes" : "No"}
               </li>
             ))}
           </ul>
           <button className="mt-4 p-2 bg-green-500 text-white rounded-lg">
             Release Milestone Payment
           </button>
         </div>
       );
     };

     export default ProjectDetails;
     ```

3. **Bid Submission Form**:
   - Create a new file `components/BidForm.tsx`:
     ```tsx
     import React, { useState } from 'react';

     const BidForm = ({ onSubmit }) => {
       const [bidAmount, setBidAmount] = useState('');
       const [completionTime, setCompletionTime] = useState('');

       const handleSubmit = (e) => {
         e.preventDefault();
         onSubmit({ bidAmount, completionTime });
       };

       return (
         <form onSubmit={handleSubmit} className="p-4 border rounded-lg shadow-md">
           <div className="mb-4">
             <label className="block text-sm font-bold mb-2">Bid Amount</label>
             <input
               type="number"
               value={bidAmount}
               onChange={(e) => setBidAmount(e.target.value)}
               className="p-2 border rounded-lg w-full"
               required
             />
           </div>
           <div className="mb-4">
             <label className="block text-sm font-bold mb-2">Completion Time (days)</label>
             <input
               type="number"
               value={completionTime}
               onChange={(e) => setCompletionTime(e.target.value)}
               className="p-2 border rounded-lg w-full"
               required
             />
           </div>
           <button type="submit" className="p-2 bg-blue-500 text-white rounded-lg">
             Submit Bid
           </button>
         </form>
       );
     };

     export default BidForm;
     ```

### Connecting to the Smart Contract

To connect your Next.js frontend to the DesignByBid smart contract:

1. **Install Web3 Dependencies**:
   - Run the following command to install `ethers` and `web3-react`:
     ```bash
     npm install ethers @web3-react/core
     ```

2. **Create a Web3 Provider**:
   - Set up a Web3 provider in `pages/_app.tsx`:
     ```tsx
     import { Web3ReactProvider } from '@web3-react/core';
     import { Web3Provider } from '@ethersproject/providers';
     import '../styles/globals.css';

     function getLibrary(provider) {
       return new Web3Provider(provider);
     }

     function MyApp({ Component, pageProps }) {
       return (
         <Web3ReactProvider getLibrary={getLibrary}>
           <Component {...pageProps} />
         </Web3ReactProvider>
       );
     }

     export default MyApp;
     ```

3. **Integrate Smart Contract Functions**:
   - Create a new file `utils/contract.ts` to handle interactions with the smart contract:
     ```tsx
     import { Contract, ethers } from 'ethers';
     import DesignByBidABI from './DesignByBidABI.json';

     // Replace with your deployed contract address
     const CONTRACT_ADDRESS = '0xYourContractAddressHere';

     export const getContract = (provider) => {
       const signer = provider.getSigner();
       return new Contract(CONTRACT_ADDRESS, DesignByBidABI, signer);
     };

     export const fetchProjects = async (provider) => {
       const contract = getContract(provider);
       const projects = await contract.getAllProjects();
       return projects.map((project) => ({
         id: project.id.toString(),
         description: project.description,
         budget: ethers.utils.formatEther(project.budget),
         deadline: project.deadline.toNumber(),
         bids: project.bids.map((bid) => ({
           bidder: bid.bidder,
           bidAmount: ethers.utils.formatEther(bid.bidAmount),
           completionTime: bid.completionTime.toNumber(),
         })),
         milestones: project.milestones,
         milestonePaid: project.milestonePaid,
       }));
     };

     export const submitBid = async (provider, projectId, bidAmount, completionTime) => {
       const contract = getContract(provider);
       const tx = await contract.submitBid(
         projectId,
         ethers.utils.parseEther(bidAmount),
         completionTime
       );
       await tx.wait();
     };

     export const releaseMilestonePayment = async (provider, projectId, milestoneIndex) => {
       const contract = getContract(provider);
       const tx = await contract.releaseMilestonePayment(projectId, milestoneIndex);
       await tx.wait();
     };
     ```

4. **Fetching Projects**:
   - Update the `pages/index.tsx` to fetch and display projects using the `fetchProjects` function:
     ```tsx
     import { useEffect, useState } from 'react';
     import { useWeb3React } from '@web3-react/core';
     import { fetchProjects } from '../utils/contract';
     import ProjectList from '../components/ProjectList';

     const Home = () => {
       const { library } = useWeb3React();
       const [projects, setProjects] = useState([]);

       useEffect(() => {
         if (library) {
           fetchProjects(library).then(setProjects);
         }
       }, [library]);

       return (
         <div className="container mx-auto">
           <h1 className="text-3xl font-bold mb-6">DesignByBid Projects</h1>
           <ProjectList projects={projects} />
         </div>
       );
     };

     export default Home;
     ```

5. **Submitting a Bid**:
   - In `components/ProjectDetails.tsx`, add a form for submitting bids:
     ```tsx
     import React from 'react';
     import { useWeb3React } from '@web3-react/core';
     import BidForm from './BidForm';
     import { submitBid } from '../utils/contract';

     const ProjectDetails = ({ project }) => {
       const { library } = useWeb3React();

       const handleBidSubmit = async ({ bidAmount, completionTime }) => {
         await submitBid(library, project.id, bidAmount, completionTime);
         // Optionally refresh project details after bidding
       };

       return (
         <div className="p-4 border rounded-lg shadow-md">
           <h2 className="text-2xl font-bold">{project.description}</h2>
           <p>Budget: {project.budget}</p>
           <p>Deadline: {new Date(project.deadline).toLocaleDateString()}</p>
           <h3 className="mt-4 text-xl font-bold">Bids:</h3>
           <ul>
             {project.bids.map((bid, index) => (
               <li key={index}>
                 Bid by {bid.bidder}: {bid.bidAmount} - Completion Time: {bid.completionTime} days
               </li>
             ))}
           </ul>

           <BidForm onSubmit={handleBidSubmit} />
         </div>
       );
     };

     export default ProjectDetails;
     ```

6. **Releasing Milestone Payments**:
   - In the same `components/ProjectDetails.tsx`, add a button to release milestone payments:
     ```tsx
     import React from 'react';
     import { useWeb3React } from '@web3-react/core';
     import { releaseMilestonePayment } from '../utils/contract';

     const ProjectDetails = ({ project }) => {
       const { library } = useWeb3React();

       const handleReleasePayment = async (milestoneIndex) => {
         await releaseMilestonePayment(library, project.id, milestoneIndex);
         // Optionally refresh project details after releasing payment
       };

       return (
         <div className="p-4 border rounded-lg shadow-md">
           <h2 className="text-2xl font-bold">{project.description}</h2>
           <p>Budget: {project.budget}</p>
           <p>Deadline: {new Date(project.deadline).toLocaleDateString()}</p>
           <h3 className="mt-4 text-xl font-bold">Bids:</h3>
           <ul>
             {project.bids.map((bid, index) => (
               <li key={index}>
                 Bid by {bid.bidder}: {bid.bidAmount} - Completion Time: {bid.completionTime} days
               </li>
             ))}
           </ul>

           <h3 className="mt-4 text-xl font-bold">Milestones:</h3>
           <ul>
             {project.milestones.map((milestone, index) => (
               <li key={index}>
                 Milestone {index + 1}: {milestone} - Paid: {project.milestonePaid[index] ? "Yes" : "No"}
                 {!project.milestonePaid[index] && (
                   <button
                     className="ml-4 p-2 bg-green-500 text-white rounded-lg"
                     onClick={() => handleReleasePayment(index)}
                   >
                     Release Payment
                   </button>
                 )}
               </li>
             ))}
           </ul>
         </div>
       );
     };

     export default ProjectDetails;
     ```

### Conclusion

By integrating blockchain technology and zkSync with the traditional Design-Bid-Build (DBB) process, we can revolutionize construction project management. The smart contract ensures transparency, reduces disputes, and automates key processes such as bid submissions and milestone payments. The Next.js frontend offers an intuitive interface for interacting with the smart contract, allowing project owners and contractors to manage projects efficiently and securely.

This project is just the beginning of what's possible when combining blockchain with construction. The future is bright, and this project is a step towards a more efficient, transparent, and automated construction industry.

