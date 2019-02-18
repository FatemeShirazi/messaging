
We consider a mix network using a sphinx-like packet format and loopix-like cover traffic, in which mix nodes are basically payed for every packet, although some secondary probabilistic layer might reduces those payments density on chain.  We shall discuss two possible underlying payment technologies available here, blind tokens and payment channels.  

# Channels vs blinded tokens

We've one major technical decision between payment channels or blinded tokens, or perhaps some hybrid.

## Blinded tokens

We can solve anonymity problems quite effectively with blind signed tokens.  Any known blinded token scheme requires the user withdraw numerous tokens and encrypt one token to each hop in the sphinx packet's routing information.  We might find some scheme that incorporates the sphinx packet's public key and reblinding operation, but this sounds tricky.

### Blind signature schemes

There are two flavours of blinded tokens worth considering:

First, bind signatures permit another party like the relay being paid to validate the token's correctness.  I believe simple blind Schnorr signatures were broken, and the fix yields a huge signature, so the obvious blind signatures schemes are RSA which sounds too large, and pairing based schemes like BLS or the Short Randomizable Signatures (SRS) by Pointcheval and Sanders.  I believe BLS gives optimal size, aggregation, and threshold properties, including that nodes can aggregate signatures from numerous packets.  I've explored SRS looking for possible advantages in integration with the packet format, but no luck so far, and I did not explore questions like aggregation.

Second, there are schemes in which only the issuer can verify the token, like oblivious PRFs or perhaps algebraic MACs.  We consider all blind signature options like BLS to be slow, but not compared to network round trips for coin depositing.  It's plausible nodes must do this traffic quickly anyways though.  I suppose this seriously complicates actually using any threshold scheme, but perhaps the verification could be designed to avoid threshold entirely.

We currently envision that BLS signatures for blind signed tokens because relays spend |issuers|+1 pairings per payment aggregation period.  We note however that payment aggregation periods should be long enough for relays to drop non-paying packets.  

We do need a traitor tracing algorithm in this aggregate BLS signature verification code too, which creates a DoS vector.  We do currently expect to need this traitor tracing algorithm anywhere in polkadot, but things change.

As an optimisation, we might explore the rich literature on batch issuing blind signed tokens, but this sounds tricky without violating the one-more forgery / known target inversion assumptions required for blind signatures.

### Issuers

We must minimize issuers because they fiber the anonymity set.  We could maybe make issuers=1 by using threshold signing, maybe a DFinity style VSS + VRF, as PVSS per signature sounds expensive.  

As a rule, we envision blind signatures schemes requiring massive stake for normal payments, so an issuer holds stake equal to its total expected payout from which all payouts occur, while all payments by customers are held by some smart contract until the coins validity expires, so that customers can claim refunds if they never spent their token.  In principle threshold blind signatures might avoid this stake.  Aside from thresholds, we could consider individual issuers with a staking scheme or even combined stake and threshold issuer pools. 

We only care about routing payments here though, so users need not necessarily be eligible for refunds.  We thus do not require such strong defenses for either funds not being unjustifiably issued or for users obtaining refunds.  It follows nicer methods should exist than stake or thresholds.  If we simply issue blind signed tokens, then issuer nodes could mint as many tokens as they like though, thus cheating the network but not users.  We could simply require that issuer nodes contain a stake significants larger than the expect network usage, or even network capacity, so that all charges can be billed to their account when the epoch ends.  At first blush, this stake sounds similar to our general purpose blind signature based payment scheme, but now network usage places an upper bound on total damages, so the stake is predictable.  Also the stake need not move until the epoch ends.  All this together eliminates most complexity from issuer payee interactions.

We must still limit the issuer nodes by requiring they win some game.  We might say tax issued tokens based on the ranking of the issuer node according to their ranking by a local VRF.  In the same vein, we might avoid stake or thresholds by making the issuer nodes rewards proportional to the number of tokens paid for minus the number of tokens redeemed, but this requires a method for counting the tokens redeemed. 

It's worth exploring the economic games here to try to avoid stake or thresholds. 

### Double spending

If we use blind signatures then spending operation reveals the blind signed message, so honest users only spend each token once.  If we want each token to only be spendable once then we must however do double spending protection, which requires collecting the messages into a database, like a semi-central authority or a ultra-simple blockchain.  This sucks.

## Payment channels

We think payment channels between hops might require less crypto per hop than blind signatures, but they raise numerous anonymity issues which we discuss here.

First, we must worry if payment channel activity leaks information about packet routes.  We discuss below a scheme that seemingly produces key material with the right cryptographic unlinkability, although this requires a proof.  We also need channels to pay for numerous packets over a long enough time frame, but that's likely an independent design goal.  I think all sounds okay but involves some future work or at least arguing with anonymity people.

We might even worry about mix-style attacks like flooding, meaning payment channels may reduce anonymity properties to that of pool-like mixes.  At the extreme, an adversary could send through enough packets to clog all payment channels except specific desirable ones, so that packets get dropped unless they follow the adversary's preferred routes.  We might defend against this attach by having sufficient funds in each payment channel, but if there are n nodes in k strata then there are n^2/k payment channels, and an adversary must target at most k n, so the attack's cost decreases super-linearly as the network grows.  We could improve available funds with some restricted routes scheme, but this reduces the attack cot proportionally too.

Second, we should ideally avoid linking on-chain identity with IP address, even at the guard node.  Some solutions include:  We could solve this by using blind signed tokens at the guard nodes, but this entails all the code complexity of both schemes.  As a rule ZCash style payments involve prohibitive computation, but guards should be long lived.  We should research the efficiency and code complexity of anonymous payment channels like BOLT vs normal payment channels, but the zero knowledge proofs involved may kill this.  Also BOLT requires a strongly anonymous ZCash style currency underneath for full protection.  We could employ monetary tokens that represents something essentially valueless, like a future statistical payment, and award them to users without users paying anything, but see notes in the probabilistic payments document (TODO: which part?).  All three solutions sound hard, but not harder than making blind signatures work, and BOLT might exist as a quasi-off the shelf solution eventually.

Third, we cannot reveal information about packet drops beyond one hop, including via the payment channel.  We discuss this more carefully below ultimately payment channels should burn any rewards not transfered onwards.  We should ensure this does not create griefing attacks via user non-payment however, but that's likely okay if we do not retrofit anything too dangerously probabilistic.  There is thus an incentive to be in earlier strata so that fewer payments get dropped by users choosing bad nodes.

Fourth, we expect all payments should know how much further they traverse through the network.  This is fine for normal traffic in a stratified mix net, but not for all cover traffic in Loopix.  A priori, this appears problematic for free route networks too, which sound easier to build, even if we prefer stratified eventually.  We might address both problems with some clever scheme for over paying or paying out packets reward tokens instead of actual "money" and having nodes convert that the "money" later.

Issues:

 - Asymptotic attacks exploiting channels as pool mixes.
 - Requires anonymous payment scheme like ZCash.
 - We must burn payments when packets get dropped, which creates incentives ot be in early strata.
 - Relays loose cover traffic loops of Loopix if we do not invent some parallel token denomination.

## Coconut

We might use a zero-knowledge proof instead of blind signature, but Alice requires considerable time making one for every hop, and the hops require some time to verify.  There are faster zero-knowledge schemes like zkSTARKs that require too much space, but maybe some simple DLEQ proof or some witness-indistinguishably trickery gives interesting compromises.  

Alice could for example use rerandomizable certificates like Coconut, which each hop reblinds.  At worst, Alice could provide a new DLEQ proof of her certificates' correctness, which costs only a couple curve points.  See: https://arxiv.org/pdf/1802.07344.pdf

We observe that Coconut's redemption blinding point kappa cannot itself be reblinded, although nu can be, so this scheme works but costs 3 thin points (48 bytes) plus 1 fat point (96 bytes), plus one NIZK per hop, along with one NIZK to withdraw.  

We have a trivial withdrawal predicate phi, so our withdrawal NIZK is a simple DLEQ proof on the thin curve, as its equations live entirely on the thin curve.  In principle implementing this NIZK should not be too much more complex than implementing say Schnorr signatures, but we might need to dig multi-scalar exponentiations out form some zero knowledge library.  All this was previously known from the underlying Short Randomizable Signatures scheme by Pointcheval and Sanders, see page 9 of https://eprint.iacr.org/2015/525.pdf  

We also have a trivial prove predicate phi', but now we have equations on both curves, so afaik a vanilla DLEQ proof does not suffice.  It's presumably still a fairly straightforward Fiat-Shamir transformation to produce a pairing-based verification that works, but now each hop requires sending two fat points, a couple scalars, doing multi-scalar exponentiations on both curves, and a pairing.  We can likely batch at least the pairing though.

We want realize essentially the same protocol with phi and phi' trivial, but with minimal per hop data and computation, so just exploring ways to massage the scheme sounds wise.  We could check if the multi-attribute variant hides non-revealed attributes securely enough.  If so, an attribute per hop avoids hiding m and thus avoids the NIZKs entirely.  We'd worry about users reusing tokens under this scheme however.  We could select some faster BN curve to improve performance, but doing so may harm anonymity, especially if we use the tokens as packet public keys.

### Users must stake tokens

If we use reblindable certificates like coconut then each token is spendable an unlimited number of times.  We can make the users rebuy the tokens periodically, but this does not tell us much about what nodes routes the most traffic.  We must also prevent users from gifting tokens to multiple friends.  If we assume all users want privacy, not an ideal assumption, then we should make reblinding the token require a secret that permits stealing the user's money.  I think coconut can give this.  I said coconut seems too slow, but you could likely batch this DLEQ verification across the pairing, but I'd need to work out how it works first.  I think the only irreparably sucky part is users needing to stake their tokens, which sounds more expensive than buying tokens.

I suspect blind Schnorr might suffer from only a minor forgery threat, not any attacks on blinding.  It's three move, but this might make it viable for something, especially if it could be turned into a certificate scheme.  This is an interesting problem, even if it may not solve our problems.


# Cover traffic

There is an advantage to not sending any cover traffic if you need to pay for cover traffic.  There is high level work that does not distinguish much between cover traffic and simply having more real traffic, but actual designs like Loopix require cover traffic.

We could maybe adapt schemes from the probabilistic payment scheme here to compel users to send cover traffic, but this sounds like much of those scheme's complexity, and penalties always suck.  In particular, users do legitimately go offline or send extra messages.

We might forget about enforcing conver traffic, and permit users to save money by running modified software, which provides the simplest solution.  We can tune conver traffic down and latency up, but this impacts user experience. 

We could make small packets free but limited and only charge for large packets, which we do not protect with cover traffic, except user might then transfer large data slowly with the small packet size, as the cover traffic volume exceeds anything else.

# Cryptographic unlinkability

Imagine some user Alice has constructed a packet header with curve point P and some fragment of the packet's path looks like
  .. -> R_0 - R_1 -> R_2 -> R_3 -> ..
If we want to use a payment channel, then we need to produce a token that only Alice and R_i know, but R_i only learns if they forward the packet to R_{i+1}.  Alice must share this token with R_{i-1} without breaking any security elsewhere.

Associated to P_i at R_i, we define a 128 bit value that R_{i+1} computes and sends back to R_i given by
  delivery_receipt = H(KEX(P,R_{i+1}) ++ "Reward Auth Out")
We could then shield delivery_receipt from R_{i+1} if R_i computes another value
  safer_delivery_receipt = AES(kex_dr, reward_auth_back)
where
  kex_dr = H(KEX(P,R_i) ++ "Delivery Receipt Safety")
If we were not using probabilistic payments, then Alice could encrypt a payment to R_i using delivery_receipt or R_{i-1} could do so using safer_delivery_receipt.

As R_{i+1} never learns safer_delivery_receipt, we could safely have Alice encrypt safer_delivery_receipt to R_{i-1}, who then encrypts a payment to Alice using it.  So safer_delivery_receipt is the desired cryptographic token. 

We do not pay some node R_i that drops traffic, but what happens to the money when a packet is dropped?

Case 1:  If the money flows back to Alice then presumably earlier nodes like R_{i-2} must learn that their payment channel pools were depleted less, which permits R_i to signal R_{i-2} and thus creates a deanonymization attack on Alice.

Case 2:  Alice appears safe if the money simply stays with R_{i-1}, but this creates an economic conflict of interest anytime two sequential nodes collude:  If R_{i-1} and R_i collude then R_i loses their money, but R_{i-1} gains the money for R_i, R_{i+1}, etc.

Instead, we expect the payment channel should simply burn any money that cannot be transfered onwards, probably by requiring an outgoing payment with specific characteristics.  We do not want the previous hop to detect this burning.

We need to spell out exactly what this payment channel that burns the excess looks like.





