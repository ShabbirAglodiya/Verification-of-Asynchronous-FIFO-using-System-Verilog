# Verification-of-Asynchronous-FIFO-using-System-Verilog

This project is a SystemVerilog-based verification environment designed to test a FIFO (First In, First Out) design. The environment consists of a series of classes that implement a generator, driver, monitor, and scoreboard, which work together to apply random stimuli to the FIFO design, capture outputs, and verify correctness.


# Synchronous FIFO Overview:
A Synchronous FIFO (First In, First Out) is a type of data structure used for buffering data between two systems operating under the same clock domain. It is commonly used in digital circuits and systems to manage data flow between different parts of a design, ensuring that data is read and written in the correct order.

## Key Features of Synchronous FIFO:
### Single Clock Domain:

In a synchronous FIFO, both the read and write operations are controlled by the same clock signal. This simplifies design by avoiding issues related to clock domain crossing.

### Fixed Data Width and Depth:

A synchronous FIFO has a defined data width (number of bits per data entry) and depth (number of data entries that can be stored). These parameters are determined at design time based on system requirements.

### Data Order Preservation:

Data written into the FIFO is stored in the order it was received and is read out in the same order, preserving the sequence of data. This makes FIFOs suitable for queue-like operations.

### Status Flags:

Synchronous FIFOs typically provide status flags such as:
Full: Indicates that the FIFO cannot accept more data until space is made available by reading.
Empty: Indicates that there is no data to read from the FIFO.
Almost Full/Almost Empty: Optional flags that provide early warnings of the FIFO nearing its full or empty state.
Simple Control Interface:

The control interface usually consists of read and write enable signals, along with status flags. This makes the FIFO easy to integrate into various digital systems.
Uses in Digital Systems:

Synchronous FIFOs are used in a variety of applications, including data buffering between processors and peripherals, rate adaptation between different parts of a system, and data queuing in communication systems.

## Typical Operation:
### Write Operation:

Data is written into the FIFO when the write enable (wr) signal is asserted and the FIFO is not full. The data is stored in the next available location.

### Read Operation:

Data is read from the FIFO when the read enable (rd) signal is asserted and the FIFO is not empty. The oldest data entry is removed from the FIFO and made available at the output.

### Status Monitoring:

The system continuously monitors the FIFO's status flags to determine when it is safe to perform read or write operations, avoiding data overflow or underflow conditions.

# Key Features:
Constrained Randomization: Generates random read and write operations with controlled probabilities.
Mailboxes for Communication: Facilitates communication between various testbench components.
Scoreboarding: Ensures that data read from the FIFO matches the data written to it.

# Components:
## Transaction:
The transaction class represents a single operation (either read or write) in the testbench. It includes various fields to model the state and data associated with an operation.

oper: Randomized bit controlling whether the transaction is a read or write.
rd, wr: Read and write control bits.
data_in, data_out: 8-bit data input and output.
full, empty: Flags indicating FIFO status.

## Generator:
The generator class creates random transactions and sends them to the driver via a mailbox. It generates a specified number of transactions based on a count parameter.

mbx: Mailbox used to send transactions to the driver.
next, done: Events to signal when to proceed and when generation is complete.

## Driver:
The driver class interacts with the DUT (Device Under Test), applying transactions received from the generator. It implements read and write operations based on the control signals in each transaction.

fif: Virtual interface to the FIFO.
reset(): Resets the DUT.
write(): Performs a write operation to the FIFO.
read(): Performs a read operation from the FIFO.

## Monitor:
The monitor class observes the interactions between the driver and the DUT, capturing relevant information and forwarding it to the scoreboard.

fif: Virtual interface to the FIFO.
mbx: Mailbox used to send captured transactions to the scoreboard.

## Scoreboard:
The scoreboard class is responsible for verifying that the FIFO behaves as expected by comparing the expected data with the actual output from the DUT.

din: Queue to store expected data values.
err: Counter for mismatched data errors.
run(): Continuously checks for mismatches and logs results.

## Environment:
The environment class instantiates and connects all the other components, ensuring they work together seamlessly to execute the testbench.

pre_test(): Prepares the DUT for testing by resetting it.
test(): Runs the main simulation, coordinating the generator, driver, monitor, and scoreboard.
post_test(): Concludes the simulation and reports error counts.

# Results:
The testbench applies a series of randomized read and write operations to the FIFO, capturing and verifying the output. The scoreboard logs any mismatches between the expected and actual data, outputting error counts at the end of the simulation.

# Output Waveforms:
The output waveforms capture the internal signals of the FIFO and testbench interactions. These waveforms are helpful for debugging and verifying the correct operation of the testbench and DUT.
