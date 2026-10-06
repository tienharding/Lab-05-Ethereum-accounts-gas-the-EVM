# Lab 05 — Ethereum accounts, gas & the EVM

**Họ tên:** Nguyen Viet Tien
---

## Lab 5.1 — Thêm TrustKeys L1 vào MetaMask

- Network name: TrustKeys L1 testnet
- RPC URL: `https://l1testnet.trustkeys.network`
- Chain ID: `11968`
- CHECK: địa chỉ trong MetaMask trùng địa chỉ dùng ở Buổi 1 (app.dem.chat) ✓

**Q1. Vì sao chainId nằm trong mọi giao dịch được ký (EIP-155)?**

Cùng một khóa bí mật cho ra cùng một địa chỉ trên mọi chuỗi EVM. Nếu chữ ký không gắn với chuỗi cụ thể, một giao dịch ký trên testnet có thể bị phát lại (replay attack) trên Sepolia hay mainnet với cùng nonce và gây mất tiền. EIP-155 đưa `chainId` vào dữ liệu được băm và ký, nên chữ ký chỉ hợp lệ trên đúng chuỗi 11968.

---

## Lab 5.2 — Đọc chuỗi bằng web3.py

Output chạy `fee_history.py`:

```
<dán output 20 dòng base fee + kết luận chain is quiet/BUSY vào đây>
```

**Q2. Vì sao base fee đứng ở đáy?**

Theo EIP-1559, `baseFee_next = baseFee × (1 + 0.125 × (gasUsed − target) / target)` với target = 15M gas. Ở đây `gasUsedRatio` ≈ 2% (khoảng 0,6M gas), thấp hơn xa mức target 50% của cap 30M, nên base fee bị kéo xuống mỗi block (tối đa −12,5%). Khi base fee chỉ còn vài wei (ví dụ 8 wei), phần giảm `8 × 0,125 × 0,96 ≈ 0,96` bị làm tròn xuống 0 bởi phép chia số nguyên, nên base fee không giảm thêm và kẹt ở đáy. Muốn nó tăng lại phải có các block liên tục đầy hơn 50%.

**Q3. Phần tử dư của `baseFeePerGas` (N+1) là gì và ví dùng thế nào?**

Phần tử cuối là base fee dự kiến của block kế tiếp (chưa được đào), tính từ `baseFee` và `gasUsed` của block mới nhất. Ví dùng nó để đặt `maxFeePerGas ≈ 2 × nextBaseFee + maxPriorityFee`: hệ số 2 là đệm vì base fee có thể tăng tối đa 12,5% mỗi block, còn `maxPriorityFee` lấy từ các percentile `reward`. Phần chênh giữa `maxFee` và giá thực tế được hoàn lại cho người gửi.

---

## Lab 5.3 — Giao dịch type-2 và phân rã phí

- Tx hash: `0x...`
- Loại giao dịch (type): 2 (EIP-1559)
- maxFeePerGas: `...` | maxPriorityFeePerGas: `...`

Output:

```
<dán dòng "paid = burned + tip ? True" vào đây>
```

| Thành phần | Giá trị (wei) |
|---|---|
| baseFee | ... |
| effectiveGasPrice | ... |
| paid | ... |
| burned | ... |
| tip | ... |

**Q4. Tăng max base fee nhưng giữ nguyên priority fee, `effectiveGasPrice` có đổi không?**

Không đổi (trong trường hợp bình thường). `effectiveGasPrice = baseFee + min(maxPriorityFee, maxFee − baseFee)`. Base fee do giao thức quyết định, còn `maxFee` chỉ là trần. Nếu `maxFee − baseFee` đã lớn hơn hoặc bằng priority fee thì `min` vẫn bằng priority fee, nên giá thực không đổi. Giá chỉ tăng khi trần cũ quá thấp đến mức đang cắt bớt tip.

**Q5. 21.000 gas đến từ đâu, vì sao giao dịch thất bại vẫn tốn gas?**

21.000 là intrinsic gas, chi phí cố định của mọi giao dịch, bao gồm xác minh chữ ký, ghi giao dịch vào chuỗi và cập nhật trạng thái (nonce, số dư). Transfer thuần không có calldata nên không cộng thêm. Giao dịch thất bại vẫn tốn gas vì node đã thực hiện công việc thật (xác minh, thực thi đến điểm lỗi, đưa vào block, tăng nonce); nếu miễn phí thì kẻ xấu có thể spam giao dịch lỗi để làm nghẽn mạng (DoS).

---

## Lab 5.4 — Sepolia Etherscan

- Tx hash: `0x705d918c787a35be126ba9c258f3f1f04c247723987c7467f4b6732f6752d5b8` (Sepolia, block 11857738, Success)
- Contract được gọi (To): `0xC959483DBa39aa9E78757139af0e9a2EDEb3f42D` (đã verify)
- Hàm được gọi: `...` (điền tên hàm hiển thị ở dòng Function trên Etherscan)
- Selector 4 byte (MethodID): `0x........` (điền theo Etherscan)
- Tham số đã giải mã:

| # | Tên | Kiểu | Giá trị |
|---|---|---|---|
| 0 | token | address | `0x94a9D9AC8a22534E3FaCa9F4e7F2E2cf85d5E4C8` |
| 1 | to | address | `0x92F4F396CC35572e3dCd07ca7f6D4a35a525B3A7` |
| 2 | amount | uint256 | `1000000000` (= 1.000 USDC với 6 chữ số thập phân) |

- Event trong Logs: `Transfer(address indexed from, address indexed to, uint256 value)`
  - Phát ra bởi token USDC `0x94a9D9AC8a22534E3FaCa9F4e7F2E2cf85d5E4C8`
  - topic0 = `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef` (keccak256 của chữ ký event)
  - from = `0x000…000` (zero address, nghĩa là token mới được tạo/mint)
  - to = `0x92F4F396CC35572e3dCd07ca7f6D4a35a525B3A7`
  - value = `1000000000`
- Nhận xét: người gọi `0x92F4…B3A7` gọi contract `0xC959…f42D`, contract này nhận địa chỉ `token` làm tham số và (nhiều khả năng) gọi sang token để mint cho `to`. Event `Transfer` ở tab Logs do contract token USDC phát ra, không phải contract được gọi trực tiếp.

**(a) Input Data đã giải mã (selector + tham số):**

![alt text](<Screenshot 2026-09-09 cau a.png>)

**(b) Tab Logs:**

![alt text](<Screenshot 2026-09-09 cau b.png>)

**Q6. EVM dùng 4 byte selector để nhảy tới đúng hàm thế nào, vì sao Etherscan cần ABI?**

Bytecode của hợp đồng mở đầu bằng một dispatcher: nó lấy 4 byte đầu của calldata, so lần lượt với selector của từng hàm (`PUSH4`/`EQ`/`JUMPI`) và nhảy tới `JUMPDEST` của hàm khớp, không khớp thì vào fallback hoặc revert. Phần calldata còn lại chỉ là các từ 32 byte thô, không chứa tên hay kiểu tham số, và selector là hàm băm một chiều, nên Etherscan cần ABI để biết tên hàm và kiểu tham số mà giải mã thành địa chỉ, số, v.v.
