# Used Goods of Tomorrow

## assignment
Come to Tommorow-Mart for all your used goods! We even have a partnership with FutureBank to grant you 500 free credeits to start! I wonder who the first millionare will be to purchase the deed to Founders' Vault?
[https://usedgoods.web.2026.sunshinectf.games/](https://usedgoods.web.2026.sunshinectf.games/)

### Initial Reconnaissance

Upon entering the site, we are presented with an e-shop displaying various items. According to the challenge description, our main goal is to acquire the "Founders' Vault," which costs a whopping 1,000,000 cr.

<img width="1418" height="2205" alt="temp3428886750605694515" src="https://github.com/user-attachments/assets/f1098f96-237b-42b0-b784-dcb75c40169f" />

The website also mentions that registering a new account grants us 500 cr. My initial thought was to check for a simple parameter tampering vulnerability, perhaps we could intercept the registration request and change our starting balance from 500 to 5,000,000, or modify the item price to 0 during checkout. To test this theory, I proceeded to register an account.
<img width="660" height="826" alt="temp7981792884919400060" src="https://github.com/user-attachments/assets/d48b1d7e-bd22-48b9-b077-711c5be04933" />

Intercepting the registration traffic in Burp Suite revealed the payload structure and showed that the application communicates with a GraphQL endpoint.

<img width="619" height="565" alt="temp607668181082846692" src="https://github.com/user-attachments/assets/35be9af7-b5cd-4c5e-bdd8-208181c9fec9" />

However, analyzing the subsequent request exchanges showed no indication that the 500 cr balance or the item prices could be easily tampered with from the client side. The validation was properly handled on the server.

### Exploring the Purchasing Logic
While examining the actual purchase process, I noticed a feature that allows users to apply promo codes, formatted similarly to SCOUT-10.

<img width="862" height="892" alt="temp9023962887503220965" src="https://github.com/user-attachments/assets/679ef6ed-ff27-4f88-b8b3-17b90c6f284f" />

Since we are dealing with a GraphQL API, I sent the previous request to Burp Repeater, utilized the built-in GraphQL tools, and attempted to send an Introspection Query.

<img width="942" height="779" alt="temp5322349301039391132" src="https://github.com/user-attachments/assets/1e950af8-5993-4217-baf8-bf605b2e3d32" />

Bingo! The schema was not protected against introspection, and the server responded with the full API structure.
### Analyzing Schema

Browsing through the leaked GraphQL schema, I found a flag field. The schema documentation hinted that the flag would be revealed when the required credit amount for the transaction is exactly 0

<img width="1010" height="597" alt="temp2646224078519774015" src="https://github.com/user-attachments/assets/9e058580-e9f7-4431-be23-1b8d78997590" />

From this, I deduced that we likely need to reduce the price of the vault to 0, which could potentially be achieved by finding a 100% discount promo code.

Digging further into the schema, I discovered a query for listing available discounts, but it required a vendorKey as an argument.

<img width="1009" height="615" alt="temp2495513546821016314" src="https://github.com/user-attachments/assets/ce64d1a8-2185-41c0-add9-65fc3895ddbf" />

Upon closer inspection of the exposed schema and available queries, I stumbled upon a piece of code (likely left in by mistake) that referenced how to obtain this vendorKey.

<img width="1014" height="542" alt="temp6664329988957136608" src="https://github.com/user-attachments/assets/df213dd4-91af-4fdc-9565-04173a7bc446" />

### Exploitation

I moved over to the GraphQL queries in Burp Suite's Site Map (obtained from schema), located the query responsible for fetching the vendor key, and sent it to Repeater to extract the key.

<img width="1012" height="722" alt="temp439042984043761966" src="https://github.com/user-attachments/assets/c1932977-8ff3-4c7f-9eec-55b56bc33fa5" />

With the vendorKey in hand, the final step was to use it in the discount query. I grabbed the discount query format from the Target tab, injected the vendor key, and executed it.

<img width="1012" height="784" alt="temp8451455217315938918" src="https://github.com/user-attachments/assets/86c29452-378b-40a8-8cd6-418324e7b46b" />

The server responded with a list of promo codes, revealing a 100% discount code: FOUNDERS-100.

Applying this code during the checkout process successfully reduced the vault's price to 0. Completing the purchase then returned the flag!

<img width="1313" height="901" alt="temp585980714655238535" src="https://github.com/user-attachments/assets/ea091136-cc1c-48c3-a295-4433e64d3c27" />
