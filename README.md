# Fake News Detection on MultiChain

A MultiChain re-implementation of the core idea of **"Fake Media Detection Based on Natural Language Processing and Blockchain Approaches"** (Z. Shahbazi and Y.-C. Byun, *IEEE Access*, vol. 9, 2021, DOI 10.1109/ACCESS.2021.3112607).

The original paper uses Hyperledger Fabric. This project maps its entities and workflow onto a basic **MultiChain** network: permissioned nodes, permissions, streams, a native asset, and round-robin mining.

## What it does

1. Only approved addresses can join the chain (permissioned network).
2. A news organization is registered, and a publisher posts a news item (text, content hash, publisher hash).
3. A simple NLP step scores the text as fake, real or suspicious.
4. Validators vote real or fake. Only validators with a credibility score of at least 50 may vote.
5. Once at least 3 votes exist, the majority decides the verdict, which is stored permanently.
6. Validators who voted with the verdict receive +10 CRED points. Validators who voted against it get -5 logged on chain.
7. Fake items are also stored in a fake-news backlog stream for future detection.

## Mapping: paper to MultiChain

| Paper (Hyperledger Fabric) | This project (MultiChain) |
|---|---|
| Permissioned network, identity manager | `anyone-can-*` set to `false`, admin grants `connect,send,receive` |
| PoA consensus, validators | Round-robin mining by permitted miners (`mine` permission, `mining-diversity`) |
| News organization | Address with `news.write`, registered in the `orgs` stream |
| Publisher / Validator | Addresses with `news.write` / `votes.write` |
| News transaction on the ledger | `news`, `votes`, `verdicts` streams |
| Credibility score | `CRED` asset plus a penalty log in the `credibility` stream |
| Fake media backlog | `fakebacklog` stream |
| Composer REST server | `multichain-cli` / JSON-RPC and `multichain-web-demo` |
| NLP | Simple word-count scorer (off-chain, result stored in `nlp` stream) |
| Reinforcement learning | Not implemented (off-chain in the paper too) |

## Repository contents

```
.
├── fakenews_multichain.ipynb   # main notebook: run top to bottom
├── fakenews.py                 # same logic as a command-line script
├── diagrams/                   # architecture and workflow figures
├── screenshots/                # screenshots used in the report
└── README.md
```

## Requirements

- Linux, WSL or Google Colab (Python 3)
- MultiChain 2.3.3 Community (`multichaind`, `multichain-cli`, `multichain-util` on the PATH)
- PHP and `php-curl` (only for `multichain-web-demo`)

## Install MultiChain (Linux / Colab)

```bash
cd /tmp
wget https://www.multichain.com/download/multichain-2.3.3.tar.gz
tar -xzf multichain-2.3.3.tar.gz
cd multichain-2.3.3
sudo mv multichaind multichain-cli multichain-util /usr/local/bin/
multichain-util        # prints help text if the install worked
```

If the download link has changed, copy the current Linux 64-bit link from https://www.multichain.com/download-community/.

## How to run

1. Open `fakenews_multichain.ipynb` and run the cells from the top.
2. The first cells create the chain and edit `params.dat` **before the first start** (the parameters are fixed after the node starts).
3. Then run: start node, setup, core functions, and the demo cells.
4. To start over, stop the node and delete `~/.multichain/fakenews`.

Command-line version:

```bash
python3 fakenews.py setup
python3 fakenews.py register-org "BBC"
python3 fakenews.py publish publisher NEWS001 "Officials said the report was confirmed by data"
python3 fakenews.py vote validator1 NEWS001 real
python3 fakenews.py vote validator2 NEWS001 real
python3 fakenews.py vote validator3 NEWS001 fake
python3 fakenews.py tally NEWS001
python3 fakenews.py scores
python3 fakenews.py show NEWS001
```

## Key `params.dat` changes

```
anyone-can-connect = false
anyone-can-send = false
anyone-can-receive = false
anyone-can-create = false
anyone-can-issue = false
anyone-can-mine = false
target-block-time = 15
```

## Demo results

| Demo | Result |
|---|---|
| A: real news (2 real votes, 1 fake vote) | Verdict REAL, validators 1 and 2 got +10 CRED, validator 3 got -5 |
| B: validator tries to post news | Blocked by MultiChain (error -704, publishing not allowed from this address) |
| C: fake news (3 fake votes) | Verdict FAKE, added to `fakebacklog` |

## multichain-web-demo and the Goffycoin task

1. Clone https://github.com/MultiChain/multichain-web-demo and copy `config-example.txt` to `config.txt`.
2. Fill in the RPC values from `~/.multichain/<chain>/multichain.conf` and `params.dat`:

   ```
   default.name=fakenews
   default.rpchost=127.0.0.1
   default.rpcport=<default-rpc-port>
   default.rpcuser=<rpcuser>
   default.rpcpassword=<rpcpassword>
   ```
3. Run `php -S localhost:8080` in the demo folder and open http://localhost:8080.
4. Issue the asset (for example `Goffycoin`) on the Issue page and send it to another address on the Send page.

**Never commit `multichain.conf`, `config.txt` or notebook outputs that contain your RPC password.**

## Limitations

- The publisher's address is visible on chain. Only the script hides it from validators (a hash is shown instead).
- Vote counting, the credibility threshold and the verdict logic run in Python, not on chain, because MultiChain has no smart contracts.
- Penalties are logged in a stream because assets cannot be taken back.
- The admin address can write to every stream and is therefore a trusted party.
- NLP is a toy word counter, and the paper's reinforcement learning model is not implemented.

## Future direction addressed

The paper's conclusion points to enhancing and investigating the Proof-of-Authority protocol and user validation. This project addresses user and validator validation with MultiChain's built-in permission model and a credibility threshold for voting.

## Reference

Z. Shahbazi and Y.-C. Byun, "Fake Media Detection Based on Natural Language Processing and Blockchain Approaches," *IEEE Access*, vol. 9, pp. 128442-128453, 2021.
