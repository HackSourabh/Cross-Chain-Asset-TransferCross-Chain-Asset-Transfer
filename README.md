# Cross-Chain-Asset-TransferCross-Chain-Asset-Transfer

  #Cross-Chain Bridge Smart Contracts
  // SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";

contract BridgeGateway is ReentrancyGuard, Ownable, Pausable {
    // Events
    event AssetLocked(
        bytes32 indexed transferId,
        address indexed sender,
        address indexed receiver,
        uint256 amount,
        uint256 timestamp
    );
    
    event AssetUnlocked(
        bytes32 indexed transferId,
        address indexed receiver,
        uint256 amount,
        uint256 timestamp
    );

    // State variables
    mapping(address => bool) public supportedTokens;
    mapping(bytes32 => bool) public processedTransfers;
    mapping(bytes32 => TransferRecord) public transfers;
    
    uint256 public nonce;
    
    struct TransferRecord {
        address sender;
        address receiver;
        uint256 amount;
        uint256 timestamp;
        bool isProcessed;
    }
    
    constructor() {
        nonce = 0;
    }
    
    // Modifier to check if token is supported
    modifier supportedToken(address token) {
        require(supportedTokens[token], "Token not supported");
        _;
    }

    // Admin functions
    function addSupportedToken(address token) external onlyOwner {
        supportedTokens[token] = true;
    }
    
    function removeSupportedToken(address token) external onlyOwner {
        supportedTokens[token] = false;
    }
    
    function pause() external onlyOwner {
        _pause();
    }
    
    function unpause() external onlyOwner {
        _unpause();
    }

    // Core bridge functions
    function lockTokens(
        address token,
        address receiver,
        uint256 amount
    ) external nonReentrant whenNotPaused supportedToken(token) returns (bytes32) {
        require(amount > 0, "Amount must be greater than 0");
        require(receiver != address(0), "Invalid receiver address");
        
        // Transfer tokens to this contract
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        
        // Generate unique transfer ID
        bytes32 transferId = generateTransferId(
            msg.sender,
            receiver,
            token,
            amount,
            nonce
        );
        nonce++;
        
        // Record the transfer
        transfers[transferId] = TransferRecord({
            sender: msg.sender,
            receiver: receiver,
            amount: amount,
            timestamp: block.timestamp,
            isProcessed: false
        });
        
        emit AssetLocked(transferId, msg.sender, receiver, amount, block.timestamp);
        return transferId;
    }
    
    function unlockTokens(
        bytes32 transferId,
        address token,
        address receiver,
        uint256 amount
    ) external nonReentrant whenNotPaused onlyOwner {
        require(!processedTransfers[transferId], "Transfer already processed");
        require(amount > 0, "Amount must be greater than 0");
        require(receiver != address(0), "Invalid receiver address");
        
        // Mark transfer as processed
        processedTransfers[transferId] = true;
        transfers[transferId].isProcessed = true;
        
        // Transfer tokens to receiver
        IERC20(token).transfer(receiver, amount);
        
        emit AssetUnlocked(transferId, receiver, amount, block.timestamp);
    }
    
    // Helper functions
    function generateTransferId(
        address sender,
        address receiver,
        address token,
        uint256 amount,
        uint256 currentNonce
    ) internal pure returns (bytes32) {
        return keccak256(
            abi.encodePacked(sender, receiver, token, amount, currentNonce)
        );
    }
    
    function getTransfer(bytes32 transferId) 
        external 
        view 
        returns (TransferRecord memory) 
    {
        return transfers[transferId];
    }
}

// Wrapped token contract for the target chain
contract WrappedToken is IERC20, Ownable {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 private _totalSupply;
    
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }
    
    function mint(address account, uint256 amount) external onlyOwner {
        _mint(account, amount);
    }
    
    function burn(address account, uint256 amount) external onlyOwner {
        _burn(account, amount);
    }
    
    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }
    
    function transfer(address recipient, uint256 amount) 
        external 
        override 
        returns (bool) 
    {
        _transfer(msg.sender, recipient, amount);
        return true;
    }
    
    function allowance(address owner, address spender) 
        external 
        view 
        override 
        returns (uint256) 
    {
        return _allowances[owner][spender];
    }
    
    function approve(address spender, uint256 amount) 
        external 
        override 
        returns (bool) 
    {
        _approve(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external override returns (bool) {
        _transfer(sender, recipient, amount);
        
        uint256 currentAllowance = _allowances[sender][msg.sender];
        require(currentAllowance >= amount, "Transfer amount exceeds allowance");
        unchecked {
            _approve(sender, msg.sender, currentAllowance - amount);
        }
        
        return true;
    }
    
    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal {
        require(sender != address(0), "Transfer from zero address");
        require(recipient != address(0), "Transfer to zero address");
        require(_balances[sender] >= amount, "Transfer amount exceeds balance");
        
        unchecked {
            _balances[sender] = _balances[sender] - amount;
            _balances[recipient] = _balances[recipient] + amount;
        }
        
        emit Transfer(sender, recipient, amount);
    }
    
    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "Mint to zero address");
        
        _totalSupply += amount;
        unchecked {
            _balances[account] = _balances[account] + amount;
        }
        
        emit Transfer(address(0), account, amount);
    }
    
    function _burn(address account, uint256 amount) internal {
        require(account != address(0), "Burn from zero address");
        require(_balances[account] >= amount, "Burn amount exceeds balance");
        
        unchecked {
            _balances[account] = _balances[account] - amount;
            _totalSupply = _totalSupply - amount;
        }
        
        emit Transfer(account, address(0), amount);
    }
    
    function _approve(
        address owner,
        address spender,
        uint256 amount
    ) internal {
        require(owner != address(0), "Approve from zero address");
        require(spender != address(0), "Approve to zero address");
        
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
}
import React, { useState, useEffect } from 'react';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Alert, AlertDescription } from "@/components/ui/alert";
import { Loader2 } from "lucide-react";

// ABI imports would go here in practice
const BRIDGE_ABI = [/* Bridge ABI would go here */];
const TOKEN_ABI = [/* Token ABI would go here */];

const BridgeInterface = () => {
  // State management
  const [account, setAccount] = useState('');
  const [amount, setAmount] = useState('');
  const [receiver, setReceiver] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const [transfers, setTransfers] = useState([]);

  // Contract addresses (would be environment variables in practice)
  const BRIDGE_ADDRESS_A = "YOUR_BRIDGE_A_ADDRESS";
  const BRIDGE_ADDRESS_B = "YOUR_BRIDGE_B_ADDRESS";
  const TOKEN_ADDRESS_A = "YOUR_TOKEN_A_ADDRESS";
  const TOKEN_ADDRESS_B = "YOUR_TOKEN_B_ADDRESS";

  // Connect wallet
  const connectWallet = async () => {
    try {
      if (window.ethereum) {
        setLoading(true);
        const accounts = await window.ethereum.request({
          method: 'eth_requestAccounts'
        });
        setAccount(accounts[0]);
        
        // Setup network change listener
        window.ethereum.on('chainChanged', () => {
          window.location.reload();
        });
        
        // Setup account change listener
        window.ethereum.on('accountsChanged', (accounts) => {
          setAccount(accounts[0] || '');
        });
        
        setError('');
      } else {
        setError('Please install MetaMask or another Web3 wallet');
      }
    } catch (err) {
      setError('Failed to connect wallet: ' + err.message);
    } finally {
      setLoading(false);
    }
  };

  // Helper function to convert amount to Wei (18 decimals)
  const toWei = (amount) => {
    const parts = amount.toString().split('.');
    let decimals = parts[1] || '';
    while (decimals.length < 18) decimals += '0';
    if (decimals.length > 18) decimals = decimals.substring(0, 18);
    const wei = parts[0] + decimals;
    return wei;
  };

  // Lock tokens
  const lockTokens = async () => {
    try {
      setLoading(true);
      setError('');
      
      // Validate input
      if (!amount || !receiver) {
        throw new Error('Please fill in all fields');
      }
      
      if (!window.ethereum) {
        throw new Error('No Web3 provider detected');
      }

      const amountWei = toWei(amount);
      
      // Approve tokens first
      const approveData = window.web3.eth.abi.encodeFunctionCall({
        name: 'approve',
        type: 'function',
        inputs: [{
          type: 'address',
          name: 'spender'
        }, {
          type: 'uint256',
          name: 'amount'
        }]
      }, [BRIDGE_ADDRESS_A, amountWei]);

      await window.ethereum.request({
        method: 'eth_sendTransaction',
        params: [{
          from: account,
          to: TOKEN_ADDRESS_A,
          data: approveData,
        }]
      });
      
      // Lock tokens
      const lockData = window.web3.eth.abi.encodeFunctionCall({
        name: 'lockTokens',
        type: 'function',
        inputs: [{
          type: 'address',
          name: 'token'
        }, {
          type: 'address',
          name: 'receiver'
        }, {
          type: 'uint256',
          name: 'amount'
        }]
      }, [TOKEN_ADDRESS_A, receiver, amountWei]);

      const txHash = await window.ethereum.request({
        method: 'eth_sendTransaction',
        params: [{
          from: account,
          to: BRIDGE_ADDRESS_A,
          data: lockData,
        }]
      });
      
      // Add to transfers list
      setTransfers(prev => [...prev, {
        id: txHash,
        sender: account,
        receiver,
        amount,
        timestamp: Date.now(),
        status: 'Pending'
      }]);
      
      // Wait for transaction confirmation
      const checkTransaction = async () => {
        const receipt = await window.ethereum.request({
          method: 'eth_getTransactionReceipt',
          params: [txHash]
        });
        
        if (receipt) {
          setTransfers(prev => 
            prev.map(t => 
              t.id === txHash 
                ? {...t, status: 'Confirmed'} 
                : t
            )
          );
        } else {
          setTimeout(checkTransaction, 3000);
        }
      };
      
      checkTransaction();
      
      setAmount('');
      setReceiver('');
    } catch (err) {
      setError('Failed to lock tokens: ' + err.message);
    } finally {
      setLoading(false);
    }
  };

  // Display transfers
  const TransfersList = () => (
    <div className="mt-6">
      <h3 className="text-lg font-medium mb-4">Recent Transfers</h3>
      <div className="space-y-2">
        {transfers.map((transfer) => (
          <Card key={transfer.id} className="p-4">
            <div className="grid grid-cols-2 gap-2 text-sm">
              <div>
                <p className="text-gray-500">From</p>
                <p className="font-mono truncate">{transfer.sender}</p>
              </div>
              <div>
                <p className="text-gray-500">To</p>
                <p className="font-mono truncate">{transfer.receiver}</p>
              </div>
              <div>
                <p className="text-gray-500">Amount</p>
                <p>{transfer.amount}</p>
              </div>
              <div>
                <p className="text-gray-500">Status</p>
                <p className={`font-medium ${
                  transfer.status === 'Confirmed' 
                    ? 'text-green-600' 
                    : 'text-yellow-600'
                }`}>
                  {transfer.status}
                </p>
              </div>
            </div>
          </Card>
        ))}
      </div>
    </div>
  );

  return (
    <div className="container mx-auto p-4 max-w-4xl">
      <Card className="mb-6">
        <CardHeader>
          <CardTitle>Cross-Chain Bridge</CardTitle>
          <CardDescription>Transfer assets between networks securely</CardDescription>
        </CardHeader>
        <CardContent>
          {!account ? (
            <Button 
              onClick={connectWallet} 
              disabled={loading}
              className="w-full"
            >
              {loading ? (
                <><Loader2 className="mr-2 h-4 w-4 animate-spin" /> Connecting...</>
              ) : (
                'Connect Wallet'
              )}
            </Button>
          ) : (
            <div className="space-y-4">
              <div>
                <p className="text-sm text-gray-500 mb-1">Connected Account</p>
                <p className="font-mono">{account}</p>
              </div>
              
              <div>
                <Input
                  type="text"
                  placeholder="Amount"
                  value={amount}
                  onChange={(e) => setAmount(e.target.value)}
                  className="mb-2"
                />
                <Input
                  type="text"
                  placeholder="Receiver Address"
                  value={receiver}
                  onChange={(e) => setReceiver(e.target.value)}
                  className="mb-4"
                />
                <Button 
                  onClick={lockTokens} 
                  disabled={loading}
                  className="w-full"
                >
                  {loading ? (
                    <><Loader2 className="mr-2 h-4 w-4 animate-spin" /> Processing...</>
                  ) : (
                    'Lock Tokens'
                  )}
                </Button>
              </div>
            </div>
          )}
          
          {error && (
            <Alert variant="destructive" className="mt-4">
              <AlertDescription>{error}</AlertDescription>
            </Alert>
          )}
          
          {account && transfers.length > 0 && <TransfersList />}
        </CardContent>
      </Card>
    </div>
  );
};

#Cross-Chain Bridge Frontend
import React, { useState, useEffect } from 'react';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Alert, AlertDescription } from "@/components/ui/alert";
import { Loader2 } from "lucide-react";

// ABI imports would go here in practice
const BRIDGE_ABI = [/* Bridge ABI would go here */];
const TOKEN_ABI = [/* Token ABI would go here */];

const BridgeInterface = () => {
  // State management
  const [account, setAccount] = useState('');
  const [amount, setAmount] = useState('');
  const [receiver, setReceiver] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const [transfers, setTransfers] = useState([]);

  // Contract addresses (would be environment variables in practice)
  const BRIDGE_ADDRESS_A = "YOUR_BRIDGE_A_ADDRESS";
  const BRIDGE_ADDRESS_B = "YOUR_BRIDGE_B_ADDRESS";
  const TOKEN_ADDRESS_A = "YOUR_TOKEN_A_ADDRESS";
  const TOKEN_ADDRESS_B = "YOUR_TOKEN_B_ADDRESS";

  // Connect wallet
  const connectWallet = async () => {
    try {
      if (window.ethereum) {
        setLoading(true);
        const accounts = await window.ethereum.request({
          method: 'eth_requestAccounts'
        });
        setAccount(accounts[0]);
        
        // Setup network change listener
        window.ethereum.on('chainChanged', () => {
          window.location.reload();
        });
        
        // Setup account change listener
        window.ethereum.on('accountsChanged', (accounts) => {
          setAccount(accounts[0] || '');
        });
        
        setError('');
      } else {
        setError('Please install MetaMask or another Web3 wallet');
      }
    } catch (err) {
      setError('Failed to connect wallet: ' + err.message);
    } finally {
      setLoading(false);
    }
  };

  // Helper function to convert amount to Wei (18 decimals)
  const toWei = (amount) => {
    const parts = amount.toString().split('.');
    let decimals = parts[1] || '';
    while (decimals.length < 18) decimals += '0';
    if (decimals.length > 18) decimals = decimals.substring(0, 18);
    const wei = parts[0] + decimals;
    return wei;
  };

  // Lock tokens
  const lockTokens = async () => {
    try {
      setLoading(true);
      setError('');
      
      // Validate input
      if (!amount || !receiver) {
        throw new Error('Please fill in all fields');
      }
      
      if (!window.ethereum) {
        throw new Error('No Web3 provider detected');
      }

      const amountWei = toWei(amount);
      
      // Approve tokens first
      const approveData = window.web3.eth.abi.encodeFunctionCall({
        name: 'approve',
        type: 'function',
        inputs: [{
          type: 'address',
          name: 'spender'
        }, {
          type: 'uint256',
          name: 'amount'
        }]
      }, [BRIDGE_ADDRESS_A, amountWei]);

      await window.ethereum.request({
        method: 'eth_sendTransaction',
        params: [{
          from: account,
          to: TOKEN_ADDRESS_A,
          data: approveData,
        }]
      });
      
      // Lock tokens
      const lockData = window.web3.eth.abi.encodeFunctionCall({
        name: 'lockTokens',
        type: 'function',
        inputs: [{
          type: 'address',
          name: 'token'
        }, {
          type: 'address',
          name: 'receiver'
        }, {
          type: 'uint256',
          name: 'amount'
        }]
      }, [TOKEN_ADDRESS_A, receiver, amountWei]);

      const txHash = await window.ethereum.request({
        method: 'eth_sendTransaction',
        params: [{
          from: account,
          to: BRIDGE_ADDRESS_A,
          data: lockData,
        }]
      });
      
      // Add to transfers list
      setTransfers(prev => [...prev, {
        id: txHash,
        sender: account,
        receiver,
        amount,
        timestamp: Date.now(),
        status: 'Pending'
      }]);
      
      // Wait for transaction confirmation
      const checkTransaction = async () => {
        const receipt = await window.ethereum.request({
          method: 'eth_getTransactionReceipt',
          params: [txHash]
        });
        
        if (receipt) {
          setTransfers(prev => 
            prev.map(t => 
              t.id === txHash 
                ? {...t, status: 'Confirmed'} 
                : t
            )
          );
        } else {
          setTimeout(checkTransaction, 3000);
        }
      };
      
      checkTransaction();
      
      setAmount('');
      setReceiver('');
    } catch (err) {
      setError('Failed to lock tokens: ' + err.message);
    } finally {
      setLoading(false);
    }
  };

  // Display transfers
  const TransfersList = () => (
    <div className="mt-6">
      <h3 className="text-lg font-medium mb-4">Recent Transfers</h3>
      <div className="space-y-2">
        {transfers.map((transfer) => (
          <Card key={transfer.id} className="p-4">
            <div className="grid grid-cols-2 gap-2 text-sm">
              <div>
                <p className="text-gray-500">From</p>
                <p className="font-mono truncate">{transfer.sender}</p>
              </div>
              <div>
                <p className="text-gray-500">To</p>
                <p className="font-mono truncate">{transfer.receiver}</p>
              </div>
              <div>
                <p className="text-gray-500">Amount</p>
                <p>{transfer.amount}</p>
              </div>
              <div>
                <p className="text-gray-500">Status</p>
                <p className={`font-medium ${
                  transfer.status === 'Confirmed' 
                    ? 'text-green-600' 
                    : 'text-yellow-600'
                }`}>
                  {transfer.status}
                </p>
              </div>
            </div>
          </Card>
        ))}
      </div>
    </div>
  );

  return (
    <div className="container mx-auto p-4 max-w-4xl">
      <Card className="mb-6">
        <CardHeader>
          <CardTitle>Cross-Chain Bridge</CardTitle>
          <CardDescription>Transfer assets between networks securely</CardDescription>
        </CardHeader>
        <CardContent>
          {!account ? (
            <Button 
              onClick={connectWallet} 
              disabled={loading}
              className="w-full"
            >
              {loading ? (
                <><Loader2 className="mr-2 h-4 w-4 animate-spin" /> Connecting...</>
              ) : (
                'Connect Wallet'
              )}
            </Button>
          ) : (
            <div className="space-y-4">
              <div>
                <p className="text-sm text-gray-500 mb-1">Connected Account</p>
                <p className="font-mono">{account}</p>
              </div>
              
              <div>
                <Input
                  type="text"
                  placeholder="Amount"
                  value={amount}
                  onChange={(e) => setAmount(e.target.value)}
                  className="mb-2"
                />
                <Input
                  type="text"
                  placeholder="Receiver Address"
                  value={receiver}
                  onChange={(e) => setReceiver(e.target.value)}
                  className="mb-4"
                />
                <Button 
                  onClick={lockTokens} 
                  disabled={loading}
                  className="w-full"
                >
                  {loading ? (
                    <><Loader2 className="mr-2 h-4 w-4 animate-spin" /> Processing...</>
                  ) : (
                    'Lock Tokens'
                  )}
                </Button>
              </div>
            </div>
          )}
          
          {error && (
            <Alert variant="destructive" className="mt-4">
              <AlertDescription>{error}</AlertDescription>
            </Alert>
          )}
          
          {account && transfers.length > 0 && <TransfersList />}
        </CardContent>
      </Card>
    </div>
  );
};

import React, { useState, useEffect } from 'react';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Alert, AlertDescription } from "@/components/ui/alert";
import { Loader2 } from "lucide-react";

// ABI imports would go here in practice
const BRIDGE_ABI = [/* Bridge ABI would go here */];
const TOKEN_ABI = [/* Token ABI would go here */];

const BridgeInterface = () => {
  // State management
  const [account, setAccount] = useState('');
  const [amount, setAmount] = useState('');
  const [receiver, setReceiver] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const [transfers, setTransfers] = useState([]);

  // Contract addresses (would be environment variables in practice)
  const BRIDGE_ADDRESS_A = "YOUR_BRIDGE_A_ADDRESS";
  const BRIDGE_ADDRESS_B = "YOUR_BRIDGE_B_ADDRESS";
  const TOKEN_ADDRESS_A = "YOUR_TOKEN_A_ADDRESS";
  const TOKEN_ADDRESS_B = "YOUR_TOKEN_B_ADDRESS";

  // Connect wallet
  const connectWallet = async () => {
    try {
      if (window.ethereum) {
        setLoading(true);
        const accounts = await window.ethereum.request({
          method: 'eth_requestAccounts'
        });
        setAccount(accounts[0]);
        
        // Setup network change listener
        window.ethereum.on('chainChanged', () => {
          window.location.reload();
        });
        
        // Setup account change listener
        window.ethereum.on('accountsChanged', (accounts) => {
          setAccount(accounts[0] || '');
        });
        
        setError('');
      } else {
        setError('Please install MetaMask or another Web3 wallet');
      }
    } catch (err) {
      setError('Failed to connect wallet: ' + err.message);
    } finally {
      setLoading(false);
    }
  };

  // Helper function to convert amount to Wei (18 decimals)
  const toWei = (amount) => {
    const parts = amount.toString().split('.');
    let decimals = parts[1] || '';
    while (decimals.length < 18) decimals += '0';
    if (decimals.length > 18) decimals = decimals.substring(0, 18);
    const wei = parts[0] + decimals;
    return wei;
  };

  // Lock tokens
  const lockTokens = async () => {
    try {
      setLoading(true);
      setError('');
      
      // Validate input
      if (!amount || !receiver) {
        throw new Error('Please fill in all fields');
      }
      
      if (!window.ethereum) {
        throw new Error('No Web3 provider detected');
      }

      const amountWei = toWei(amount);
      
      // Approve tokens first
      const approveData = window.web3.eth.abi.encodeFunctionCall({
        name: 'approve',
        type: 'function',
        inputs: [{
          type: 'address',
          name: 'spender'
        }, {
          type: 'uint256',
          name: 'amount'
        }]
      }, [BRIDGE_ADDRESS_A, amountWei]);

      await window.ethereum.request({
        method: 'eth_sendTransaction',
        params: [{
          from: account,
          to: TOKEN_ADDRESS_A,
          data: approveData,
        }]
      });
      
      // Lock tokens
      const lockData = window.web3.eth.abi.encodeFunctionCall({
        name: 'lockTokens',
        type: 'function',
        inputs: [{
          type: 'address',
          name: 'token'
        }, {
          type: 'address',
          name: 'receiver'
        }, {
          type: 'uint256',
          name: 'amount'
        }]
      }, [TOKEN_ADDRESS_A, receiver, amountWei]);

      const txHash = await window.ethereum.request({
        method: 'eth_sendTransaction',
        params: [{
          from: account,
          to: BRIDGE_ADDRESS_A,
          data: lockData,
        }]
      });
      
      // Add to transfers list
      setTransfers(prev => [...prev, {
        id: txHash,
        sender: account,
        receiver,
        amount,
        timestamp: Date.now(),
        status: 'Pending'
      }]);
      
      // Wait for transaction confirmation
      const checkTransaction = async () => {
        const receipt = await window.ethereum.request({
          method: 'eth_getTransactionReceipt',
          params: [txHash]
        });
        
        if (receipt) {
          setTransfers(prev => 
            prev.map(t => 
              t.id === txHash 
                ? {...t, status: 'Confirmed'} 
                : t
            )
          );
        } else {
          setTimeout(checkTransaction, 3000);
        }
      };
      
      checkTransaction();
      
      setAmount('');
      setReceiver('');
    } catch (err) {
      setError('Failed to lock tokens: ' + err.message);
    } finally {
      setLoading(false);
    }
  };

  // Display transfers
  const TransfersList = () => (
    <div className="mt-6">
      <h3 className="text-lg font-medium mb-4">Recent Transfers</h3>
      <div className="space-y-2">
        {transfers.map((transfer) => (
          <Card key={transfer.id} className="p-4">
            <div className="grid grid-cols-2 gap-2 text-sm">
              <div>
                <p className="text-gray-500">From</p>
                <p className="font-mono truncate">{transfer.sender}</p>
              </div>
              <div>
                <p className="text-gray-500">To</p>
                <p className="font-mono truncate">{transfer.receiver}</p>
              </div>
              <div>
                <p className="text-gray-500">Amount</p>
                <p>{transfer.amount}</p>
              </div>
              <div>
                <p className="text-gray-500">Status</p>
                <p className={`font-medium ${
                  transfer.status === 'Confirmed' 
                    ? 'text-green-600' 
                    : 'text-yellow-600'
                }`}>
                  {transfer.status}
                </p>
              </div>
            </div>
          </Card>
        ))}
      </div>
    </div>
  );

  return (
    <div className="container mx-auto p-4 max-w-4xl">
      <Card className="mb-6">
        <CardHeader>
          <CardTitle>Cross-Chain Bridge</CardTitle>
          <CardDescription>Transfer assets between networks securely</CardDescription>
        </CardHeader>
        <CardContent>
          {!account ? (
            <Button 
              onClick={connectWallet} 
              disabled={loading}
              className="w-full"
            >
              {loading ? (
                <><Loader2 className="mr-2 h-4 w-4 animate-spin" /> Connecting...</>
              ) : (
                'Connect Wallet'
              )}
            </Button>
          ) : (
            <div className="space-y-4">
              <div>
                <p className="text-sm text-gray-500 mb-1">Connected Account</p>
                <p className="font-mono">{account}</p>
              </div>
              
              <div>
                <Input
                  type="text"
                  placeholder="Amount"
                  value={amount}
                  onChange={(e) => setAmount(e.target.value)}
                  className="mb-2"
                />
                <Input
                  type="text"
                  placeholder="Receiver Address"
                  value={receiver}
                  onChange={(e) => setReceiver(e.target.value)}
                  className="mb-4"
                />
                <Button 
                  onClick={lockTokens} 
                  disabled={loading}
                  className="w-full"
                >
                  {loading ? (
                    <><Loader2 className="mr-2 h-4 w-4 animate-spin" /> Processing...</>
                  ) : (
                    'Lock Tokens'
                  )}
                </Button>
              </div>
            </div>
          )}
          
          {error && (
            <Alert variant="destructive" className="mt-4">
              <AlertDescription>{error}</AlertDescription>
            </Alert>
          )}
          
          {account && transfers.length > 0 && <TransfersList />}
        </CardContent>
      </Card>
    </div>
  );
};

export default BridgeInterface;
