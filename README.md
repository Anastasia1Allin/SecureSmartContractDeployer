# SecureSmartContractDeployer
A smart contract deployment tool that ensures secure transfer and storage of assets using Ethereum's Solidity. It integrates a multi-signature wallet mechanism to provide added security.
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SecureWallet {
    address[] public owners;
    uint public requiredApprovals;
    mapping(address => bool) public isOwner;
    struct Transaction {
        address to;
        uint amount;
        bool executed;
        uint approvals;
    }
    Transaction[] public transactions;
    mapping(uint => mapping(address => bool)) public approvals;

    event Deposit(address indexed sender, uint amount);
    event TransactionCreated(uint indexed txIndex, address indexed to, uint amount);
    event TransactionApproved(uint indexed txIndex, address indexed owner);
    event TransactionExecuted(uint indexed txIndex);

    constructor(address[] memory _owners, uint _requiredApprovals) {
        require(_owners.length > 0, "Owners required");
        require(_requiredApprovals > 0 && _requiredApprovals <= _owners.length, "Invalid number of approvals");

        for (uint i = 0; i < _owners.length; i++) {
            address owner = _owners[i];
            require(owner != address(0), "Invalid owner address");
            require(!isOwner[owner], "Owner already added");
            isOwner[owner] = true;
            owners.push(owner);
        }
        requiredApprovals = _requiredApprovals;
    }

    receive() external payable {
        emit Deposit(msg.sender, msg.value);
    }

    function createTransaction(address _to, uint _amount) external onlyOwner {
        transactions.push(Transaction({to: _to, amount: _amount, executed: false, approvals: 0}));
        emit TransactionCreated(transactions.length - 1, _to, _amount);
    }

    function approveTransaction(uint _txIndex) external onlyOwner {
        require(_txIndex < transactions.length, "Invalid transaction index");
        require(!approvals[_txIndex][msg.sender], "Transaction already approved");
        require(!transactions[_txIndex].executed, "Transaction already executed");

        approvals[_txIndex][msg.sender] = true;
        transactions[_txIndex].approvals++;

        emit TransactionApproved(_txIndex, msg.sender);

        if (transactions[_txIndex].approvals >= requiredApprovals) {
            executeTransaction(_txIndex);
        }
    }

    function executeTransaction(uint _txIndex) internal {
        require(transactions[_txIndex].approvals >= requiredApprovals, "Not enough approvals");
        require(!transactions[_txIndex].executed, "Transaction already executed");

        transactions[_txIndex].executed = true;
        payable(transactions[_txIndex].to).transfer(transactions[_txIndex].amount);

        emit TransactionExecuted(_txIndex);
    }

    modifier onlyOwner() {
        require(isOwner[msg.sender], "Not an owner");
        _;
    }
}
