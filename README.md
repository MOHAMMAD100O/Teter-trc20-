// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract EuroToCrypto is ReentrancyGuard, Ownable {
    IUniswapV2Router02 public uniswapRouter;
    address public EURT;  // Euro token (e.g., EURT on BSC)
    address public USDT;  // Destination token (USDT or any other BEP20 token)

    // MetaMask wallet address on BSC network (stored securely)
    bytes32 private recipientAddressHash;  // Store hash of the recipient address

    constructor(address _uniswapRouter, address _EURT, address _USDT, address _recipientAddress) {
        uniswapRouter = IUniswapV2Router02(_uniswapRouter);
        EURT = _EURT;
        USDT = _USDT;
        recipientAddressHash = keccak256(abi.encodePacked(_recipientAddress));  // Store hash of address
    }

    // Convert Euro to USDT and send to MetaMask address
    function convertAndSend(uint256 amountIn) public onlyOwner nonReentrant {
        // Validate recipient address hash
        address recipientAddress = address(uint160(uint256(recipientAddressHash)));
        require(recipientAddress != address(0), "Invalid recipient address");

        require(IERC20(EURT).balanceOf(address(this)) >= amountIn, "Insufficient balance");

        // Validate allowance before proceeding
        uint256 allowance = IERC20(EURT).allowance(address(this), address(uniswapRouter));
        require(allowance >= amountIn, "Allowance too low");

        // Approve Uniswap to spend the Euro token (EURT)
        IERC20(EURT).approve(address(uniswapRouter), amountIn);

        address;
        path[0] = EURT;  // Euro token (EURT)
        path[1] = USDT;  // USDT (or other crypto token)

        uint256 initialBalance = IERC20(USDT).balanceOf(address(this));

        // Perform the swap on Uniswap
        uniswapRouter.swapExactTokensForTokens(
            amountIn,         // Amount of Euro to convert
            0,                // Minimum amount of USDT (set to 0 for market order)
            path,             // Conversion path
            address(this),    // Keep the resulting tokens in the contract
            block.timestamp + 300  // Deadline (5 minutes)
        );

        uint256 newBalance = IERC20(USDT).balanceOf(address(this));
        require(newBalance > initialBalance, "Swap failed");

        uint256 amountReceived = newBalance - initialBalance;
        IERC20(USDT).transfer(recipientAddress, amountReceived);
    }

    // Deposit Euro into the contract
    function depositFunds(uint256 amount) public onlyOwner {
        IERC20(EURT).transferFrom(msg.sender, address(this), amount);
    }

    // Withdraw all USDT from the contract
    function withdrawAll() public onlyOwner nonReentrant {
        uint256 balance = IERC20(USDT).balanceOf(address(this));
        require(balance > 0, "No balance to withdraw");
        IERC20(USDT).transfer(recipientAddress, balance); // Send USDT to MetaMask address
    }

    // Allow the owner to update the recipient address
    function updateRecipientAddress(address newRecipient) public onlyOwner {
        recipientAddressHash = keccak256(abi.encodePacked(newRecipient));  // Update hash of recipient address
    }
}
