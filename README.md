# base3ccimport time
from collections import defaultdict, Counter
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20
DOMINANCE_THRESHOLD = 0.50  # 50% of activity


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect to Base RPC")

    print("Connected to Base")
    print("Detecting dominant wallet activity per token...\n")

    last_block = w3.eth.block_number

    while True:
        try:
            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                from_block = current_block - WINDOW_BLOCKS
                to_block = current_block

                logs = w3.eth.get_logs({
                    "fromBlock": from_block,
                    "toBlock": to_block,
                    "topics": [TRANSFER_TOPIC]
                })

                token_wallet_activity = defaultdict(Counter)

                for log in logs:

                    token = log["address"]

                    from_addr = decode_address(log["topics"][1])
                    to_addr = decode_address(log["topics"][2])

                    token_wallet_activity[token][from_addr] += 1
                    token_wallet_activity[token][to_addr] += 1

                print(f"\nBlocks {from_block} → {to_block}")

                for token, counter in token_wallet_activity.items():

                    total = sum(counter.values())

                    if total == 0:
                        continue

                    top_wallet, top_count = counter.most_common(1)[0]

                    dominance = top_count / total

                    if dominance >= DOMINANCE_THRESHOLD:

                        print("⚠️ Dominant Wallet Activity")
                        print("Token:", token)
                        print("Wallet:", top_wallet)
                        print("Dominance:", round(dominance * 100, 2), "%")
                        print("Interactions:", top_count)
                        print("Total activity:", total)
                        print()

                last_block = current_block

            time.sleep(3)

        except Exception as e:
            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
