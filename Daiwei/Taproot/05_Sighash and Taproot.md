# Sighash and Taproot

## Introduction

​	In order to ensure that no one spends the holder's BTC without permission, a signature is required to complete a BTC spend. Since a transaction may have multiple inputs and outputs, there are also multiple types of signatures. SIGHASH is a type of flag used to indicate which part of the transaction is signed. By choosing different SIGHASH, different application scenarios can be effectively dealt with.

## Application

![sighash_1](https://github.com/AAweidai/PictureBed/blob/master/taproot/sighash_1.png?raw=1)

​	Although the Bitcoin network only defines four values, SIGHASH_ANYONECANPAY can be combined with the first three values by bitwise |, resulting in a total of six possible values for SIGHASH. As shown in the figure above, there are 6 different combinations of digital signatures that can be added to the transaction. It is worth noting that different inputs of a transaction can use different SIGHASH flags to achieve a complex combination of spending conditions.

![sighash_1](https://github.com/AAweidai/PictureBed/blob/master/taproot/sighash_2.png?raw=1)

1. SIGHASH_ALL (0x01): This is the default setting of every user's wallet that we know. It signs all inputs and outputs of each transaction, and any changes to the transaction will invalidate the signature. This is essentially "I only agree to use bitcoins whose input and output must be accurate, and the address can be received". This is a commonly used transaction signature type for each of us to create a specific output by spending a specific input.

2. SIGHASH_NONE (0x02): This will sign all inputs of the transaction, but not any outputs. In fact, it will produce an authorization phrase: "I can participate in this transaction, but I don't particularly care where it goes." On the surface, this seems insecure, and it seems that it is a waste of money without signing any output. Indeed, if you create a tx with only one input and sign it with SIGHASH_NONE, then miners will be able to simply change the output to the output they control. This structure is equivalent to creating a "blank check" or "blank check" of a specific amount. It promises input, but allows the output locking script to be changed. Anyone can write their Bitcoin address into the output lock script and exchange transactions. The available application scenario is to satisfy the situation of "I agree to spend my money, as long as everyone else also spends it". It is expected that one of the other signers will subsequently use the SIGHASH_ALL transaction to ensure all the output of the transaction and send the payment to the mutually agreed output set.

3. SIGHASH_SINGLE (0x03): This signature signs all inputs and a corresponding output. The corresponding output is the output with the same index as the signature (that is, if your input is vin 0, the output you want to sign must be vout 0). This actually means: "As long as the money reaches this address, I agree to participate in all these entered transactions." The available application scenario is that A and B need to pay someone 1.5 BTC, and A contributes 1 BTC. B contributes 0.5 BTC. Therefore, A can use SIGHASH_SINGLE to create a transaction of 1 BTC input and one 1.5 BTC output to the person who needs to pay. Then, B can add output for its change address and use SIGHASH_ALL or SIGHASH_SINGLE to complete the transaction.

4. SIGHASH_ALL | SIGHASH_ANYONECANPAY (0x81): Similar to SIGHASH_ALL, and sign all outputs. However, it only signs one input. Essentially, "As long as the following recipients receive these payments, I agree to participate in this transaction. I don't care about any other inputs in this transaction. The available application scenario is that this structure can be used for "crowdfunding" transactions, People trying to raise funds can construct a transaction with a single output, which pays the "target" amount to the crowdfunding promoter. Such a transaction is obviously invalid because it has no input. But now others can add themselves They use ALL | ANYONECANPAY to sign their input. Unless enough input is collected to achieve the value of the output, the transaction is invalid. Each donation is a "collateral" until the entire goal is raised. The fundraiser can only collect the amount.

5. SIGHASH_NONE | SIGHASH_ANYONECANPAY (0x82): Similar to SIGHASH_NONE, but only enter a signature in it. This is essentially like saying: "I can send this BTC. In fact, I don't even care if it is sent in this transaction. This is a signed note saying that any transaction including this transaction can spend this BTC ". The available application scenarios do not provide any guarantees and are only used as proof of costs. Such a signature allows anyone to define an input and include that input in any other signature with arbitrary output. By making such a signature, BTC can be spent without any control.

6. SIGHASH_SINGLE | SIGHASH_ANYONECANPAY (0x83): Similar to SIGHASH_SINGLE, except that it only signs the input that contains it and the corresponding output. This means "I absolutely want to move so many BTC to this output, but I don't care about any other inputs and outputs in this transaction". The available application scenario is A to B transfer colored coins:

   A creates a transaction with one input and one output:

   - Input 0: 1 colored coin from A

   - Output 0: 1 bitcoin to A 

   B sees A's transaction and can add transaction:

   - Input 1: 1 bitcoin from B

   - Output 1: 1 colored coin to B

   Essentially, this allows the input of 1 colored coin held by A to be signed and output to its controlled address with 1 BTC. Then, A publishes this invalid transaction. The reason for the invalidity is that there is no input of 1 BTC. Finally, anyone who wants to buy 1 BTC of dyed coins can add an input greater than or equal to 1 BTC, and an output that requires 1 dyed coin to complete the transaction.


## Sighash and Taproot

​	In bip118, it is proposed to introduce a new type of SIGHASH called SIGHASH_ANYPREVOUT. This SIGHASH describes a new type of public key for tapscript (BIP 342) transactions. It allows the signature of these public keys to not promise the exact UTXO spent. That is, by deleting the previous output (and optional witness script and value), it is allowed to dynamically rebind the signed transaction to another previous output that requires the same key authorization. Moreover, this SIGHASH flag is renamed from SIGHASH_NOINPUT to reflect that although any prevout may be used with the signature, some aspects of the input are still submitted, that is, the input of the nSequence value, as well as the spending conditions and amount.

## Conclusion

​	In general, SIGHASH is a signature type mark. Using different signature types, we can choose to sign different parts of the input and output of the transaction, so as to effectively complete the cost of BTC in different scenarios.