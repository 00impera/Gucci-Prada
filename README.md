// TON Blockchain Integration for LUXCHAIN
// This script handles wallet connection, transactions, and blockchain interactions

// Initialize TON SDK and TonWeb
let wallet = null;
let tonweb = null;
let contractAddress = "EQA..."; // Replace with actual smart contract address
let connected = false;

// DOM Elements
document.addEventListener('DOMContentLoaded', function() {
  // Connect elements to variables
  const connectWalletBtn = document.getElementById('connectWallet');
  const joinPresaleBtn = document.getElementById('joinPresale');
  const walletBalanceDiv = document.getElementById('walletBalance');
  const tonBalanceSpan = document.getElementById('tonBalance');
  const presaleForm = document.getElementById('presaleForm');
  const amountInput = document.getElementById('amount');
  const receiveInput = document.getElementById('receive');
  const buyButton = document.getElementById('buyButton');
  const blockchainData = document.getElementById('blockchainData');
  const walletModal = document.querySelector('.wallet-modal');
  
  // Presale Stats Elements
  const raisedAmount = document.getElementById('raisedAmount');
  const participantCount = document.getElementById('participantCount');
  const countdown = document.getElementById('countdown');
  const presaleProgress = document.getElementById('presaleProgress');

  // Blockchain Data Elements
  const userWalletAddress = document.getElementById('userWalletAddress');
  const userTonBalance = document.getElementById('userTonBalance');
  const userLxcBalance = document.getElementById('userLxcBalance');
  const userContribution = document.getElementById('userContribution');
  const userLxcAllocation = document.getElementById('userLxcAllocation');
  const contributionStatus = document.getElementById('contributionStatus');
  const transactionHistory = document.getElementById('transactionHistory');

  // Event Listeners
  connectWalletBtn.addEventListener('click', toggleWalletConnection);
  joinPresaleBtn.addEventListener('click', scrollToPresale);
  amountInput.addEventListener('input', calculateTokenAmount);
  buyButton.addEventListener('click', participateInPresale);

  // Initialize
  updatePresaleStats();
  startCountdown();

  // Check if TON wallet extension is available
  function checkTonWalletAvailability() {
    if (typeof window.TON !== 'undefined') {
      console.log('TON wallet extension detected');
      return true;
    } else {
      console.log('TON wallet extension not detected');
      showNotification('Please install TON wallet extension to connect.', 'warning');
      return false;
    }
  }

  // Toggle wallet connection
  async function toggleWalletConnection() {
    if (!connected) {
      // Connect wallet
      if (!checkTonWalletAvailability()) {
        showWalletModal();
        return;
      }

      try {
        // Initialize TON SDK
        tonweb = new TonWeb(new TonWeb.HttpProvider('https://toncenter.com/api/v2/jsonRPC'));
        
        // Request wallet connection
        const provider = window.TON;
        const walletInfo = await provider.requestWallets();
        
        if (walletInfo && walletInfo.length > 0) {
          wallet = walletInfo[0];
          connected = true;
          
          // Update UI
          connectWalletBtn.textContent = 'Disconnect';
          walletBalanceDiv.style.display = 'block';
          buyButton.textContent = 'Buy LXC Tokens';
          buyButton.disabled = false;
          blockchainData.style.display = 'block';
          
          // Get wallet balance
          await updateWalletBalance();
          
          // Update blockchain data
          updateBlockchainData();
          
          showNotification('Wallet connected successfully!', 'success');
        }
      } catch (error) {
        console.error('Failed to connect wallet:', error);
        showNotification('Failed to connect wallet. Please try again.', 'error');
      }
    } else {
      // Disconnect wallet
      wallet = null;
      connected = false;
      
      // Update UI
      connectWalletBtn.textContent = 'Connect Wallet';
      walletBalanceDiv.style.display = 'none';
      buyButton.textContent = 'Connect Wallet to Participate';
      buyButton.disabled = true;
      blockchainData.style.display = 'none';
      
      showNotification('Wallet disconnected.', 'success');
    }
  }

  // Update wallet balance
  async function updateWalletBalance() {
    if (wallet && connected) {
      try {
        // Get TON balance
        const balance = await tonweb.getBalance(wallet.address);
        const tonBalance = TonWeb.utils.fromNano(balance);
        
        // Update UI
        tonBalanceSpan.textContent = parseFloat(tonBalance).toFixed(4);
        userTonBalance.textContent = parseFloat(tonBalance).toFixed(4);
      } catch (error) {
        console.error('Failed to get wallet balance:', error);
      }
    }
  }

  // Update blockchain data
  async function updateBlockchainData() {
    if (wallet && connected) {
      try {
        // Display wallet address
        userWalletAddress.textContent = wallet.address.slice(0, 6) + '...' + wallet.address.slice(-4);
        
        // Get LXC balance (requires contract interaction)
        // This is a placeholder, actual implementation depends on the contract
        const lxcBalance = await getLxcBalance(wallet.address);
        userLxcBalance.textContent = lxcBalance;
        
        // Get user contribution
        const contribution = await getUserContribution(wallet.address);
        userContribution.textContent = contribution.amount;
        userLxcAllocation.textContent = contribution.amount * 5000; // 1 TON = 5000 LXC
        contributionStatus.textContent = contribution.status;
        
        // Get transaction history
        const transactions = await getTransactionHistory(wallet.address);
        updateTransactionHistory(transactions);
      } catch (error) {
        console.error('Failed to update blockchain data:', error);
      }
    }
  }

  // Calculate token amount based on TON input
  function calculateTokenAmount() {
    const amount = parseFloat(amountInput.value) || 0;
    const tokenAmount = amount * 5000; // 1 TON = 5000 LXC
    receiveInput.value = `${tokenAmount.toLocaleString()} LXC`;
  }

  // Participate in presale
  async function participateInPresale() {
    if (!connected) {
      showNotification('Please connect your wallet first.', 'warning');
      return;
    }
    
    const amount = parseFloat(amountInput.value) || 0;
    
    if (amount < 5) {
      showNotification('Minimum contribution is 5 TON.', 'warning');
      return;
    }
    
    if (amount > 500) {
      showNotification('Maximum contribution is 500 TON.', 'warning');
      return;
    }
    
    try {
      // Create transaction
      const transaction = {
        to: contractAddress,
        value: TonWeb.utils.toNano(amount.toString()),
        dataType: 'text',
        data: 'buy_lxc',
      };
      
      // Send transaction
      const provider = window.TON;
      const result = await provider.sendTransaction(transaction);
      
      if (result.success) {
        showNotification('Transaction submitted successfully!', 'success');
        
        // Add transaction to history
        const newTx = {
          id: result.txid,
          amount: amount,
          date: new Date().toISOString(),
          status: 'pending'
        };
        
        // Update UI
        addTransactionToHistory(newTx);
        
        // Update blockchain data after transaction
        setTimeout(updateBlockchainData, 5000);
        setTimeout(updateWalletBalance, 5000);
      } else {
        showNotification('Transaction failed. Please try again.', 'error');
      }
    } catch (error) {
      console.error('Failed to send transaction:', error);
      showNotification('Failed to send transaction. Please try again.', 'error');
    }
  }

  // Scroll to presale section
  function scrollToPresale() {
    const presaleSection = document.querySelector('.presale');
    presaleSection.scrollIntoView({ behavior: 'smooth' });
  }

  // Show wallet modal
  function showWalletModal() {
    const modal = document.createElement('div');
    modal.className = 'wallet-modal active';
    modal.innerHTML = `
      <div class="modal-content">
        <div class="modal-header">
          <h3>Connect Wallet</h3>
          <button class="close-modal">&times;</button>
        </div>
        <div class="wallet-options">
          <div class="wallet-option" id="tonkeeperOption">
            <img src="/api/placeholder/60/60" alt="TON Keeper">
            <h4>TON Keeper</h4>
          </div>
          <div class="wallet-option" id="tonhubOption">
            <img src="/api/placeholder/60/60" alt="Tonhub">
            <h4>Tonhub</h4>
          </div>
          <div class="wallet-option" id="chromeOption">
            <img src="/api/placeholder/60/60" alt="Chrome Extension">
            <h4>TON Extension</h4>
          </div>
        </div>
        <p style="text-align: center;">Please install a TON wallet to continue</p>
      </div>
    `;
    
    document.body.appendChild(modal);
    
    // Add event listeners
    modal.querySelector('.close-modal').addEventListener('click', () => {
      document.body.removeChild(modal);
    });
    
    modal.querySelectorAll('.wallet-option').forEach(option => {
      option.addEventListener('click', () => {
        if (option.id === 'tonkeeperOption') {
          window.open('https://tonkeeper.com', '_blank');
        } else if (option.id === 'tonhubOption') {
          window.open('https://tonhub.com', '_blank');
        } else if (option.id === 'chromeOption') {
          window.open('https://chrome.google.com/webstore/detail/ton-wallet/nphplpgoakhhjchkkhmiggakijnkhfnd', '_blank');
        }
        
        document.body.removeChild(modal);
      });
    });
  }

  // Update presale stats
  function updatePresaleStats() {
    // This would typically come from the contract
    const totalRaised = 375000;
    const hardCap = 500000;
    const participants = 2875;
    
    raisedAmount.textContent = `${totalRaised.toLocaleString()} / ${hardCap.toLocaleString()} TON`;
    participantCount.textContent = participants.toLocaleString();
    
    // Update progress bar
    const progressPercentage = (totalRaised / hardCap) * 100;
    presaleProgress.style.width = `${progressPercentage}%`;
  }

  // Start countdown timer
  function startCountdown() {
    // Set end date to May 15, 2025
    const endDate = new Date('2025-05-15T23:59:59').getTime();
    
    const timer = setInterval(() => {
      const now = new Date().getTime();
      const distance = endDate - now;
      
      if (distance <= 0) {
        clearInterval(timer);
        countdown.textContent = 'ENDED';
        buyButton.disabled = true;
        buyButton.textContent = 'Presale Ended';
        return;
      }
      
      // Calculate time units
      const days = Math.floor(distance / (1000 * 60 * 60 * 24));
      const hours = Math.floor((distance % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
      const minutes = Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60));
      const seconds = Math.floor((distance % (1000 * 60)) / 1000);
      
      // Format countdown
      countdown.textContent = `${days}:${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
    }, 1000);
  }

  // Add transaction to history UI
  function addTransactionToHistory(tx) {
    const row = document.createElement('tr');
    
    row.innerHTML = `
      <td>${tx.id.slice(0, 6)}...${tx.id.slice(-4)}</td>
      <td>${tx.amount} TON</td>
      <td>${new Date(tx.date).toLocaleString()}</td>
      <td><span class="transaction-status status-${tx.status}">${tx.status}</span></td>
    `;
    
    transactionHistory.prepend(row);
  }

  // Update transaction history UI
  function updateTransactionHistory(transactions) {
    transactionHistory.innerHTML = '';
    
    transactions.forEach(tx => {
      addTransactionToHistory(tx);
    });
  }

  // Show notification
  function showNotification(message, type) {
    const notification = document.createElement('div');
    notification.className = `notification ${type}`;
    
    const icon = type === 'success' ? 'check-circle' : 
                type === 'error' ? 'times-circle' : 'exclamation-triangle';
    
    notification.innerHTML = `
      <i class="fas fa-${icon}"></i>
      <span>${message}</span>
    `;
    
    document.body.appendChild(notification);
    
    // Show notification
    setTimeout(() => {
      notification.classList.add('show');
    }, 100);
    
    // Hide and remove notification
    setTimeout(() => {
      notification.classList.remove('show');
      setTimeout(() => {
        document.body.removeChild(notification);
      }, 300);
    }, 5000);
  }

  // Mock functions for contract interactions
  // These would be replaced with actual contract calls
  
  async function getLxcBalance(address) {
    // Mock function for demo purposes
    return '0';
  }
  
  async function getUserContribution(address) {
    // Mock function for demo purposes
    return {
      amount: 0,
      status: 'Not Started'
    };
  }
  
  async function getTransactionHistory(address) {
    // Mock function for demo purposes
    return [];
  }
});

// TON Contract Methods

// Function to create TON contract instance
function createContract() {
  if (!tonweb) return null;
  
  // This is a placeholder. Actual contract definition depends on your contract structure
  const LuxchainContract = {
    // Contract methods would be defined here
    getBalance: async function(address) {
      // Implementation depends on your contract
      return 0;
    },
    
    buyTokens: async function(amount) {
      // Implementation depends on your contract
      return {
        success: true,
        txid: 'mock-tx-id'
      };
    }
  };
  
  return LuxchainContract;
}

// Function to load TonWeb library
function loadTonWeb() {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/tonweb/0.0.60/tonweb.js';
    script.onload = () => resolve();
    script.onerror = (error) => reject(error);
    document.head.appendChild(script);
  });
}

// Initialize on page load
window.addEventListener('DOMContentLoaded', async function() {
  try {
    await loadTonWeb();
    console.log('TonWeb loaded successfully');
  } catch (error) {
    console.error('Failed to load TonWeb:', error);
  }
});
// This is a representation of how the FunC contract for LUXCHAIN would look
// Note: This is a TypeScript pseudocode of what would be written in FunC for TON

// LUXCHAIN Token Contract for TON Blockchain
// Implements the Jetton (TON's token standard) interface

import {
  Cell,
  Address,
  StateInit,
  contractAddress,
  beginCell,
  Contract,
  ContractProvider,
  Sender,
  SendMode,
  TupleBuilder,
  Builder
} from 'ton-core';

// Token Constants
const TOKEN_NAME = "LUXCHAIN";
const TOKEN_SYMBOL = "LXC";
const TOKEN_DECIMALS = 9;
const TOTAL_SUPPLY = 1000000000n * 10n ** BigInt(TOKEN_DECIMALS); // 1 billion tokens
const TOKEN_DESCRIPTION = "Luxury fashion-themed cryptocurrency on TON";

// Error Codes
enum ErrorCodes {
  ACCESS_DENIED = 100,
  INSUFFICIENT_FUNDS = 101,
  INVALID_AMOUNT = 102,
  PRESALE_NOT_ACTIVE = 103,
  MAX_CONTRIBUTION_EXCEEDED = 104,
  MIN_CONTRIBUTION_NOT_MET = 105,
  PRESALE_FINISHED = 106
}

// Token Allocation
const AIRDROP_ALLOCATION = TOTAL_SUPPLY * 40n / 100n;    // 40%
const PRESALE_ALLOCATION = TOTAL_SUPPLY * 10n / 100n;    // 10%
const ECOSYSTEM_ALLOCATION = TOTAL_SUPPLY * 30n / 100n;  // 30%
const TEAM_ALLOCATION = TOTAL_SUPPLY * 20n / 100n;       // 20%

// Presale Config
interface PresaleConfig {
  startTime: number;         // Unix timestamp when presale starts
  endTime: number;           // Unix timestamp when presale ends
  rate: bigint;              // How many tokens per TON
  minContribution: bigint;   // Minimum TON contribution
  maxContribution: bigint;   // Maximum TON contribution
  hardCap: bigint;           // Maximum TON to raise
  totalRaised: bigint;       // Total TON raised so far
  participantCount: number;  // Number of participants
}

export class LuxchainToken implements Contract {
  constructor(
    readonly address: Address,
    readonly init?: StateInit
  ) {}

  // Create the initial contract
  static createFromConfig(
    owner: Address,
    presaleWallet: Address,
    teamWallet: Address,
    ecosystemWallet: Address,
    code: Cell
  ) {
    // Create the initial data cell for the contract
    const data = beginCell()
      .storeAddress(owner)                // Contract owner
      .storeAddress(presaleWallet)        // Presale wallet
      .storeAddress(teamWallet)           // Team wallet 
      .storeAddress(ecosystemWallet)      // Ecosystem wallet
      .storeCoins(TOTAL_SUPPLY)           // Total supply
      .storeDict(null)                    // Balances dictionary
      .storeRef(beginCell()               // Presale config
        .storeUint(Math.floor(Date.now() / 1000), 32)             // Start time (now)
        .storeUint(Math.floor(Date.now() / 1000) + 30 * 24 * 3600, 32)  // End time (30 days from now)
        .storeCoins(5000n)                // Rate: 1 TON = 5000 LXC
        .storeCoins(5 * 10n ** 9n)        // Min contribution: 5 TON
        .storeCoins(500 * 10n ** 9n)      // Max contribution: 500 TON
        .storeCoins(500000 * 10n ** 9n)   // Hard cap: 500,000 TON
        .storeCoins(0)                    // Total raised: 0 TON
        .storeUint(0, 32)                 // Participant count: 0
        .endCell()
      )
      .storeBuffer(Buffer.from(TOKEN_NAME))       // Token name
      .storeBuffer(Buffer.from(TOKEN_SYMBOL))     // Token symbol
      .storeUint(TOKEN_DECIMALS, 8)               // Token decimals
      .storeBuffer(Buffer.from(TOKEN_DESCRIPTION)) // Token description
      .endCell();
    
    const init = { code, data };
    return new LuxchainToken(contractAddress(0, init), init);
  }

  // Get contract methods
  async getTokenInfo(provider: ContractProvider): Promise<{
    name: string;
    symbol: string;
    decimals: number;
    totalSupply: bigint;
    description: string;
  }> {
    const result = await provider.get('get_token_info', []);
    return {
      name: result.stack.readString(),
      symbol: result.stack.readString(),
      decimals: result.stack.readNumber(),
      totalSupply: result.stack.readBigNumber(),
      description: result.stack.readString()
    };
  }

  // Get balance of an address
  async getBalance(provider: ContractProvider, address: Address): Promise<bigint> {
    const result = await provider.get('get_balance', [
      { type: 'slice', cell: beginCell().storeAddress(address).endCell() }
    ]);
    return result.stack.readBigNumber();
  }

  // Get presale info
  async getPresaleInfo(provider: ContractProvider): Promise<PresaleConfig> {
    const result = await provider.get('get_presale_info', []);
    return {
      startTime: result.stack.readNumber(),
      endTime: result.stack.readNumber(),
      rate: result.stack.readBigNumber(),
      minContribution: result.stack.readBigNumber(),
      maxContribution: result.stack.readBigNumber(),
      hardCap: result.stack.readBigNumber(),
      totalRaised: result.stack.readBigNumber(),
      participantCount: result.stack.readNumber()
    };
  }

  // Get user contribution
  async getUserContribution(provider: ContractProvider, address: Address): Promise<{
    amount: bigint;
    tokens: bigint;
  }> {
    const result = await provider.get('get_user_contribution', [
      { type: 'slice', cell: beginCell().storeAddress(address).endCell() }
    ]);
    return {
      amount: result.stack.readBigNumber(),
      tokens: result.stack.readBigNumber()
    };
  }

  // Transfer tokens
  async transfer(
    provider: ContractProvider,
    sender: Sender,
    to: Address,
    amount: bigint
  ) {
    await provider.internal(sender, {
      value: "0.05", // 0.05 TON for gas
      sendMode: SendMode.PAY_GAS_SEPARATELY,
      body: beginCell()
        .storeUint(0x10, 32) // op: transfer
        .storeAddress(to)
        .storeCoins(amount)
        .endCell()
    });
  }

  // Participate in presale
  async participateInPresale(
    provider: ContractProvider,
    sender: Sender,
    value: bigint
  ) {
    await provider.internal(sender, {
      value: value.toString(),
      sendMode: SendMode.PAY_GAS_SEPARATELY,
      body: beginCell()
        .storeUint(0x11, 32) // op: participate_in_presale
        .endCell()
    });
  }

  // Only for owner: withdraw TON from presale
  async withdrawPresaleFunds(
    provider: ContractProvider,
    sender: Sender,
    amount: bigint
  ) {
    await provider.internal(sender, {
      value: "0.05", // 0.05 TON for gas
      sendMode: SendMode.PAY_GAS_SEPARATELY,
      body: beginCell()
        .storeUint(0x12, 32) // op: withdraw_presale_funds
        .storeCoins(amount)
        .endCell()
    });
  }

  // Only for owner: finalize presale
  async finalizePresale(
    provider: ContractProvider,
    sender: Sender
  ) {
    await provider.internal(sender, {
      value: "0.05", // 0.05 TON for gas
      sendMode: SendMode.PAY_GAS_SEPARATELY,
      body: beginCell()
        .storeUint(0x13, 32) // op: finalize_presale
        .endCell()
    });
  }

  // Only for owner: start airdrop
  async startAirdrop(
    provider: ContractProvider,
    sender: Sender,
    participants: Address[],
    amounts: bigint[]
  ) {
    // Create a cell with participants and amounts
    const participantsCell = beginCell();
    for (let i = 0; i < participants.length; i++) {
      participantsCell
        .storeAddress(participants[i])
        .storeCoins(amounts[i]);
    }

    await provider.internal(sender, {
      value: "0.05", // 0.05 TON for gas
      sendMode: SendMode.PAY_GAS_SEPARATELY,
      body: beginCell()
        .storeUint(0x14, 32) // op: start_airdrop
        .storeRef(participantsCell.endCell())
        .endCell()
    });
  }

  // Only for owner: release team tokens (after lock period)
  async releaseTeamTokens(
    provider: ContractProvider,
    sender: Sender
  ) {
    await provider.internal(sender, {
      value: "0.05", // 0.05 TON for gas
      sendMode: SendMode.PAY_GAS_SEPARATELY,
      body: beginCell()
        .storeUint(0x15, 32) // op: release_team_tokens
        .endCell()
    });
  }
}

// In the actual contract implementation, the following functions would be
// defined in FunC, TON's smart contract language:

/**
 * Here's how some core functions would be implemented in FunC:
 * 
 * recv_internal(int msg_value, cell msg_cell, slice msg_body) {
 *   // Parse incoming message
 *   int op = msg_body~load_uint(32);
 *   
 *   // Handle different operations
 *   if (op == 0x10) { // transfer
 *     handle_transfer(msg_value, msg_cell, msg_body);
 *   } else if (op == 0x11) { // participate_in_presale
 *     handle_presale_participation(msg_value, msg_cell, msg_body);
 *   } else if (op == 0x12) { // withdraw_presale_funds
 *     handle_withdraw_presale_funds(msg_value, msg_cell, msg_body);
 *   } else if (op == 0x13) { // finalize_presale
 *     handle_finalize_presale(msg_value, msg_cell, msg_body);
 *   } else if (op == 0x14) { // start_airdrop
 *     handle_start_airdrop(msg_value, msg_cell, msg_body);
 *   } else if (op == 0x15) { // release_team_tokens
 *     handle_release_team_tokens(msg_value, msg_cell, msg_body);
 *   } else {
 *     throw(666); // Unknown operation
 *   }
 * }
 * 
 * // Handle presale participation
 * handle_presale_participation(int msg_value, cell msg_cell, slice msg_body) {
 *   // Load contract data
 *   slice ds = get_data().begin_parse();
 *   
 *   // Check if presale is active
 *   cell presale_config_cell = ds~load_ref();
 *   slice presale_config = presale_config_cell.begin_parse();
 *   
 *   int start_time = presale_config~load_uint(32);
 *   int end_time = presale_config~load_uint(32);
 *   
 *   int current_time = now();
 *   throw_if(PRESALE_NOT_ACTIVE, current_time < start_time);
 *   throw_if(PRESALE_FINISHED, current_time > end_time);
 *   
 *   // Check if contribution meets requirements
 *   int rate = presale_config~load_coins();
 *   int min_contribution = presale_config~load_coins();
 *   int max_contribution = presale_config~load_coins();
 *   int hard_cap = presale_config~load_coins();
 *   int total_raised = presale_config~load_coins();
 *   int participant_count = presale_config~load_uint(32);
 *   
 *   throw_if(MIN_CONTRIBUTION_NOT_MET, msg_value < min_contribution);
 *   throw_if(MAX_CONTRIBUTION_EXCEEDED, msg_value > max_contribution);
 *   
 *   // Calculate tokens to be allocated
 *   int tokens_to_allocate = msg_value * rate;
 *   
 *   // Update total raised and participant count
 *   total_raised += msg_value;
 *   participant_count += 1;
 *   
 *   // Update user balance in the dictionary
 *   cell balances_dict = ds~load_dict();
 *   slice sender_addr = msg_sender();
 *   
 *   // Look up if user already has a balance
 *   (int found, int existing_balance) = balances_dict.udict_get?(267, sender_addr);
 *   int new_balance = found ? existing_balance + tokens_to_allocate : tokens_to_allocate;
 *   
 *   // Update the balance
 *   balances_dict~udict_set(267, sender_addr, new_balance);
 *   
 *   // Save updated contract data
 *   set_data(
 *     begin_cell()
 *     .store_slice(ds)
 *     .store_dict(balances_dict)
 *     .store_ref(
 *       begin_cell()
