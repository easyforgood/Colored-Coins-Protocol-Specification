The new colored coins protocol defines a mini scripting language to efficiently encode the flow of assets from inputs to outputs.
Each Transfers instruction is built out of 5 pieces of data (3 booleans and two integers):

## Transfer Instructions

|Command            |Type   |Memory |Meaning|
|:-----------------:|-------|:---:|-------|
|[Skip](#skip)      |Boolean|1 bit  |0 => Stay on input after processing<br/>1 => skip to next input after processing|
|[Range](#range)    |Boolean|1 bit   |0 => **Output** size is 5 bits and understood literally<br/>1 => **Output** size is 13 bits and understood as specifying a range|
|[Percent](#percent)|Boolean|1 bit   |0 => **Amount** is understood literally<br/>1 => **Amount** is understood as percent (1 byte)
|[Output](#output)  |Integer|5 bits (Range=0)<hr/>13 bits (Range=1)| Specific Output index between 0..31<hr/>(**Range**=1) Range of outputs between index 0 and the specified index (maximum 8191)
|[Amount](#amount)|Integer|[1-7](Number Encoding) bytes (Percent=0)<hr/>1 Byte (Percent=1)| Number of units to be transferred <hr/> Percent of units to be transferred


### Skip
Decides whether the **next** instruction will be processed against that same input (skip = 0) or against the next input (skip = 1).

### Range
Decides whether the [Output](#output) integer specifies the actual index of an output or the maximal index in a range of outputs (starting from index 0) as the target for the transfer instruction.

` Not yet supported in current version (0x02)`

### Percent
Decides whether the [Amount](#amount) integer should be understood as a number of units to be transferred (percent = 0) or as a percent of existing units to be transferred (percent = 1).

`Not yet supported in current version (0x02)`

### Output
Specify the target output (or range of outputs) of the transfer.

* Range = 0: Make the transfer to the output of the specified index. 
Uses 5 bits, thus allowing to specify any output index between 0 and 31. 
* Range = 1: Make the **same transfer** to **all outputs** up to the specified index.
Uses 13 bits, thus allowing to specify output ranges as big as 0-8191. Useful for Mass activities such as a sending a Christmas present coupon to all the employees in a company.  

### Amount
Specifies the amount of units (or percentage of units if percent = 1) to be transferred. 
* Percent = 0: Amount is a signed integer encoded in our [precision encoding scheme](Number Encoding) and requiring 1-7 bytes of memory, depending on the number.
* Percent = 1: The integer represents percents (with some precision after the decimal point).

#### Note on Memory Consumption
Note that the total memory requirement for encoding the Skip+Range+Percent+Output of each transfer instruction is exactly 1 or 2 bytes (depending on whether Range is 0 or 1). Therefore, each transfer instruction requires between 2 to 9 bytes, depending on the Amount.

## Transfer Rules

The rules by which the colored coins software transfers assets between inputs and outputs when parsing a valid asset transfer transaction are as follows:
* Begin with the first asset of the first input in the transaction (order is determined recursively) 
* **Range = 0**: move the specified amount of the currently processed asset to the output in the index specified in *Output*. The maximal index is 31.<br> 
A transfer instruction amount can exceed the remaining amount of an asset currently processed, if the asset is *aggregatable* - in that case, the processing will try to satisfy the remaining amount of the current transfer instruction with the next asset if such exists and of the same asset ID.
Otherwise, and in case it fails to satisfy the instruction, it is an invalid instruction and all assets are sent to the last output.<br>
After transalating the skip mechanism to an input index in each transfer instruction, applying transfers should follow this pseudo-code:
```
function transfer (tx, transferInstructions) {
	currentInput = 0, lastInput = -1, currentAsset = 0, lastAsset = -1, lastTransferInstruction = -1

	for (i = 0 ; i < transferInstructions.length ; i++) {

		if (Max(currentInput, transferInstructions[i].input) >= tx.vin.length || transferInstructions[i].output >= tx.vout.length) {
			transferAllAssetsToLastOutput(tx)
			return			
		}

		if (currrentInput < transferInstructions[i].input) {
			currentInput = transferInstructions[i].input
			currentAsset = 0
		}

		if (tx.vin[currentInput].assets[currentAsset].length == 0) {
			transferAllAssetsToLastOutput(tx)
			return		
		}

		if (lastTransferInstruction == i && (tx.vin[currentInput].assets[currentAsset].aggregationPolicy != 'aggregatable' || tx.vin[currentInput].assets[currentAsset].assetId != tx.vin[lastInput].assets[lastAsset].assetId)) {
			transferAllAssetsToLastOutput(tx)
			return			
		}

		transferFromInputToOutput(tx, transferInstructions[i], transferInstructions[i].input, currentAsset, transferInstructions[i].output, Min(transferInstructions[i].amount, tx.vin[currentInput].assets[currentAsset].amount))
		lastInput = currentInput
		lastAsset = currentAsset
		lastTransferInstruction = i
                
		if (transferInstructions[i].amount > tx.vin[currentInput].assets[currentAsset].amount) {
	  	    currentAsset++
	        while (currentInput < tx.vin.length && currentAsset > tx.vin[currentInput].assets.length - 1) {
	          currentAsset = 0
	          currentInput++
    	    }
  		    i--		// stay on the same transfer instruction
	  	}
	}
}

function transferFromInputToOutput(tx, transferInstruction, input, assetIndex, output, amount) {
	tx.vout[output].assets.concat(tx.vin[input].assets[assetIndex])
	tx.vin[input].assets[assetIndex].amount -= amount
	transferInstruction.amount -= amount
}

function transferAllAssetsToLastOutput(tx) {
	for (i = 0 ; i < tx.vout.length ; i++) {
		transfer.vout[output].assets = []
	}

	for (i = 0 ; i < tx.vin.length ; i++) {
		for (j = 0 ; j < tx.vin[i].assets.length ; j++) {
			tx.vout[tx.vout.length - 1].assets.concat(tx.vin[i].assets[j])
			tx.vin[i].assets[j].amount = 0
		}
	}
}
```
* **Range = 1**: move the specified amount of the currently processed asset to all outputs in the range of indices from 0 all the way to the index specified in *Output*. The maximal index is 8191.
* All assets with remaining amounts are automatically transferred to the last output.

## Bitcoin requirements
* A minimal dust amount in real satoshis is sent to each output for validity
* A transaction fee is left for miners to finance the processing of the transaction

## Examples
* We have 3 assets of types `A, B, C`
* The first input in our transaction has **10** units of asset `A` and **13** units of asset `B`
* The second input has **100** units of asset `C`

The following chain of instructions
```
 0 0 0 0 3    0 0 0 1 7    0 0 1 1 50    1 0 0 2 3    0 1 0 49 2
```
means:
 
1. `0 0 0 0 3` Transfer 3 units of asset `A` from the first input to the first output (index 0). **Stay** on first input.
1. `0 0 0 1 7` Transfer the remaining 7 units of asset `A` from the first input to the second output (index 1). **Stay** on first input.
1. `0 0 1 1 50` Transfer 6 units of asset `B` to second output. 6 units because 6 is 50% of 12 and we are now processing asset `B` because the previous instructions exhausted asset `A` in the first input. **Stay** on first input.
1. `1 0 0 2 3` Transfer 3 units of asset `B` to the third output (index 2). **Skip** to second input.
1. `0 1 0 49 2` Transfer 2 units of asset `C` from the second input to each output, starting from the first output all the way to the 50th output (index 49).
1. 1 unit of asset `B` is implicitly transferred to the last output, since it had 7 units and only 6 were transferred explicitly.