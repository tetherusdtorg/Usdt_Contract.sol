# Usdt_Contract.sol
Usd Credit System
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

/**
 * @title USD Tether Token (USDT)
 * @dev TRC20/ERC20 standardına uygun stablecoin
 */
contract USD_Tether {
    string public name = "USD Tether";
    string public symbol = "USDT";
    uint8 public decimals = 6;
    uint256 public totalSupply;
    uint256 private constant MAX_SUPPLY = 1000000000000000000000 * (10 ** 6);

    address public owner;
    mapping(address => uint256) public balanceOf;
    mapping(address => bool) public isBanned;
    mapping(address => mapping(address => uint256)) public allowance;

    bool public isSwapEnabled = true;
    bool public paused = false;
    uint256 public transferLimit = 1e6;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Ban(address indexed account, bool isBanned);
    event Swap(address indexed from, address indexed to, uint256 value);
    event LiquidityAdded(address indexed provider, uint256 value);
    event Mint(address indexed to, uint256 value);
    event DestroyBlackFunds(address indexed account, uint256 amount);
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    event Pause();
    event Unpause();

    modifier onlyOwner() {
        require(msg.sender == owner, "Not the contract owner");
        _;
    }

    modifier swapEnabled() {
        require(isSwapEnabled, "Swapping is disabled");
        _;
    }

    modifier whenNotPaused() {
        require(!paused, "Transfers are paused");
        _;
    }

    constructor() {
        owner = msg.sender;
        totalSupply = MAX_SUPPLY;
        balanceOf[msg.sender] = MAX_SUPPLY;
        emit Transfer(address(0), msg.sender, MAX_SUPPLY);
    }

    function transfer(address to, uint256 amount) public swapEnabled whenNotPaused returns (bool) {
        require(!isBanned[msg.sender], "Sender is banned");
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        require(!isBanned[to], "Recipient is banned");
        require(amount <= transferLimit, "Transfer exceeds limit");

        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) public swapEnabled whenNotPaused returns (bool) {
        require(!isBanned[from], "Sender is banned");
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(!isBanned[to], "Recipient is banned");
        require(amount <= allowance[from][msg.sender], "Allowance exceeded");

        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function increaseApproval(address spender, uint256 addedValue) public returns (bool) {
        allowance[msg.sender][spender] += addedValue;
        emit Approval(msg.sender, spender, allowance[msg.sender][spender]);
        return true;
    }

    function decreaseApproval(address spender, uint256 subtractedValue) public returns (bool) {
        if (subtractedValue >= allowance[msg.sender][spender]) {
            allowance[msg.sender][spender] = 0;
        } else {
            allowance[msg.sender][spender] -= subtractedValue;
        }
        emit Approval(msg.sender, spender, allowance[msg.sender][spender]);
        return true;
    }

    function mint(address to, uint256 amount) public onlyOwner {
        require(totalSupply + amount <= MAX_SUPPLY, "Exceeds maximum supply");
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Mint(to, amount);
    }

    function setBanStatus(address account, bool banStatus) public onlyOwner {
        isBanned[account] = banStatus;
        emit Ban(account, banStatus);
    }

    function toggleSwapStatus() public onlyOwner {
        isSwapEnabled = !isSwapEnabled;
    }

    function setTransferLimit(uint256 limit) public onlyOwner {
        require(limit > 0, "Transfer limit must be greater than zero");
        transferLimit = limit;
    }

    function addLiquidity(uint256 amount) public onlyOwner {
        require(amount > 0, "Amount must be greater than zero");
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");

        balanceOf[msg.sender] -= amount;
        balanceOf[address(this)] += amount;
        emit LiquidityAdded(msg.sender, amount);
    }

    function swap(address to, uint256 amount) public swapEnabled whenNotPaused {
        require(!isBanned[msg.sender], "Sender is banned");
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        require(!isBanned[to], "Recipient is banned");
        require(amount <= transferLimit, "Swap exceeds limit");

        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Swap(msg.sender, to, amount);
    }

    function pause() public onlyOwner {
        paused = true;
        emit Pause();
    }

    function unpause() public onlyOwner {
        paused = false;
        emit Unpause();
    }

    function destroyBlackFunds(address account) public onlyOwner {
        require(isBanned[account], "Address is not banned");
        uint256 amount = balanceOf[account];
        balanceOf[account] = 0;
        totalSupply -= amount;
        emit DestroyBlackFunds(account, amount);
    }

    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "New owner is zero address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
}
