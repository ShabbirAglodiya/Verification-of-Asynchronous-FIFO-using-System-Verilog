// Define the transaction class to represent FIFO transactions.
class transaction;
  
  rand bit oper;          // Randomized bit for operation control (1 for write, 0 for read)
  bit rd, wr;             // Read and write control bits
  bit [7:0] data_in;      // 8-bit data input
  bit full, empty;        // Flags indicating the full and empty status of the FIFO
  bit [7:0] data_out;     // 8-bit data output

  // Constraint to ensure equal probability of generating read and write operations
  constraint oper_ctrl {  
    oper dist {1 :/ 50 , 0 :/ 50};  // 50% probability for both read and write operations
  }
  
endclass

///////////////////////////////////////////////////

// Generator class generates a sequence of FIFO transactions.
class generator;
  
  transaction tr;           // Transaction object for generating and sending transactions
  mailbox #(transaction) mbx;  // Mailbox for communicating transactions to other components
  int count = 0;            // Number of transactions to generate
  int i = 0;                // Iteration counter for tracking the number of generated transactions
  
  event next;               // Event to signal when to send the next transaction
  event done;               // Event to signal completion of all requested transactions
   
  // Constructor for the generator class, initializing the mailbox
  function new(mailbox #(transaction) mbx);
    this.mbx = mbx;
    tr = new();  // Create a new transaction object
  endfunction; 
 
  // Task to generate and send transactions
  task run(); 
    repeat (count) begin  // Generate transactions 'count' number of times
      assert (tr.randomize) else $error("Randomization failed");  // Randomize the transaction
      i++;
      mbx.put(tr);  // Send the transaction to the mailbox
      $display("[GEN] : Oper : %0d iteration : %0d", tr.oper, i);  // Display the operation type and iteration number
      @(next);  // Wait for the 'next' event before sending the next transaction
    end
    -> done;  // Signal that all transactions are done
  endtask
  
endclass

////////////////////////////////////////////

// Driver class sends transactions to the DUT (Device Under Test).
class driver;
  
  virtual fifo_if fif;     // Virtual interface to interact with the FIFO
  mailbox #(transaction) mbx;  // Mailbox to receive transactions from the generator
  transaction datac;       // Transaction object for communication with the DUT
 
  // Constructor for the driver class, initializing the mailbox
  function new(mailbox #(transaction) mbx);
    this.mbx = mbx;
  endfunction; 
 
  // Task to reset the DUT
  task reset();
    fif.rst <= 1'b1;  // Assert the reset signal
    fif.rd <= 1'b0;   // Deassert read control
    fif.wr <= 1'b0;   // Deassert write control
    fif.data_in <= 0; // Clear data input
    repeat (5) @(posedge fif.clock);  // Wait for a few clock cycles
    fif.rst <= 1'b0;  // Deassert the reset signal
    $display("[DRV] : DUT Reset Done");
    $display("------------------------------------------");
  endtask
   
  // Task to write data to the FIFO
  task write();
    @(posedge fif.clock);  // Wait for a positive edge of the clock
    fif.rst <= 1'b0;
    fif.rd <= 1'b0;
    fif.wr <= 1'b1;  // Assert write control
    fif.data_in <= $urandom_range(1, 10);  // Generate random data
    @(posedge fif.clock);  // Wait for the next clock edge
    fif.wr <= 1'b0;  // Deassert write control
    $display("[DRV] : DATA WRITE  data : %0d", fif.data_in);  
    @(posedge fif.clock);
  endtask
  
  // Task to read data from the FIFO
  task read();  
    @(posedge fif.clock);  // Wait for a positive edge of the clock
    fif.rst <= 1'b0;
    fif.rd <= 1'b1;  // Assert read control
    fif.wr <= 1'b0;
    @(posedge fif.clock);  // Wait for the next clock edge
    fif.rd <= 1'b0;  // Deassert read control
    $display("[DRV] : DATA READ");  
    @(posedge fif.clock);
  endtask
  
  // Task to apply random stimulus to the DUT
  task run();
    forever begin
      mbx.get(datac);  // Retrieve a transaction from the mailbox
      if (datac.oper == 1'b1)  // If operation is write
        write();
      else  // If operation is read
        read();
    end
  endtask
  
endclass

///////////////////////////////////////////////////////

// Monitor class observes and logs the FIFO behavior.
class monitor;
 
  virtual fifo_if fif;     // Virtual interface to interact with the FIFO
  mailbox #(transaction) mbx;  // Mailbox for communicating transactions to the scoreboard
  transaction tr;          // Transaction object for monitoring and logging
  
  // Constructor for the monitor class, initializing the mailbox
  function new(mailbox #(transaction) mbx);
    this.mbx = mbx;     
  endfunction;
 
  // Task to continuously monitor the FIFO
  task run();
    tr = new();  // Create a new transaction object
    
    forever begin
      repeat (2) @(posedge fif.clock);  // Wait for two clock cycles
      tr.wr = fif.wr;  // Capture the write control signal
      tr.rd = fif.rd;  // Capture the read control signal
      tr.data_in = fif.data_in;  // Capture the data input
      tr.full = fif.full;  // Capture the full flag
      tr.empty = fif.empty;  // Capture the empty flag
      @(posedge fif.clock);  // Wait for the next clock edge
      tr.data_out = fif.data_out;  // Capture the data output
    
      mbx.put(tr);  // Send the captured transaction to the mailbox
      $display("[MON] : Wr:%0d rd:%0d din:%0d dout:%0d full:%0d empty:%0d", tr.wr, tr.rd, tr.data_in, tr.data_out, tr.full, tr.empty);
    end
    
  endtask
  
endclass

/////////////////////////////////////////////////////

// Scoreboard class verifies the FIFO behavior against expected results.
class scoreboard;
  
  mailbox #(transaction) mbx;  // Mailbox to receive monitored transactions
  transaction tr;          // Transaction object for storing monitored data
  event next;              // Event to signal the next check
  bit [7:0] din[$];        // Queue to store written data
  bit [7:0] temp;          // Temporary storage for data comparison
  int err = 0;             // Error counter for mismatch detection
  
  // Constructor for the scoreboard class, initializing the mailbox
  function new(mailbox #(transaction) mbx);
    this.mbx = mbx;     
  endfunction;
 
  // Task to continuously verify the FIFO transactions
  task run();
    forever begin
      mbx.get(tr);  // Retrieve a transaction from the mailbox
      $display("[SCO] : Wr:%0d rd:%0d din:%0d dout:%0d full:%0d empty:%0d", tr.wr, tr.rd, tr.data_in, tr.data_out, tr.full, tr.empty);
      
      if (tr.wr == 1'b1) begin  // If the operation is write
        if (tr.full == 1'b0) begin  // If the FIFO is not full
          din.push_front(tr.data_in);  // Store the data in the queue
          $display("[SCO] : DATA STORED IN QUEUE :%0d", tr.data_in);
        end
        else begin
          $display("[SCO] : FIFO is full");  // Indicate that the FIFO is full
        end
        $display("--------------------------------------"); 
      end
    
      if (tr.rd == 1'b1) begin  // If the operation is read
        if (tr.empty == 1'b0) begin  // If the FIFO is not empty
          temp = din.pop_back();  // Retrieve the expected data from the queue
          
          if (tr.data_out == temp)  // Compare expected and actual data
            $display("[SCO] : DATA MATCH");
          else begin
            $error("[SCO] : DATA MISMATCH");  // Report a mismatch
            err++;  // Increment error count
          end
        end
        else begin
          $display("[SCO] : FIFO IS EMPTY");  // Indicate that the FIFO is empty
        end
        
        $display("--------------------------------------"); 
      end
      
      -> next;  // Signal the next check
    end
  endtask
  
endclass

///////////////////////////////////////////////////////

// Define the environment class which encapsulates all components of the testbench
class environment;

  // Declare instances of the testbench components
  generator gen;       // Generator for creating transactions
  driver drv;          // Driver to send transactions to the DUT (Device Under Test)
  monitor mon;         // Monitor to observe the DUT behavior
  scoreboard sco;      // Scoreboard to track and verify the results
  mailbox #(transaction) gdmbx;  // Mailbox for communication between Generator and Driver
  mailbox #(transaction) msmbx;  // Mailbox for communication between Monitor and Scoreboard
  event nextgs;                 // Event used for synchronization between generator and scoreboard
  virtual fifo_if fif;          // Virtual interface to connect with the FIFO DUT
  
  // Constructor for the environment class
  function new(virtual fifo_if fif);
    gdmbx = new();        // Initialize mailbox for generator and driver communication
    gen = new(gdmbx);     // Instantiate generator with the mailbox
    drv = new(gdmbx);     // Instantiate driver with the mailbox
    msmbx = new();        // Initialize mailbox for monitor and scoreboard communication
    mon = new(msmbx);     // Instantiate monitor with the mailbox
    sco = new(msmbx);     // Instantiate scoreboard with the mailbox
    this.fif = fif;       // Assign the virtual interface
    drv.fif = this.fif;   // Connect driver to the virtual interface
    mon.fif = this.fif;   // Connect monitor to the virtual interface
    gen.next = nextgs;    // Connect generator's next event
    sco.next = nextgs;    // Connect scoreboard's next event
  endfunction
  
  // Pre-test task to perform any initialization required before the test
  task pre_test();
    drv.reset();          // Reset the driver
  endtask
  
  // Main test task to run the testbench components in parallel
  task test();
    fork
      gen.run();          // Start the generator
      drv.run();          // Start the driver
      mon.run();          // Start the monitor
      sco.run();          // Start the scoreboard
    join_any               // Wait for any of the forked tasks to complete
  endtask
  
  // Post-test task to finalize the test and display results
  task post_test();
    wait(gen.done.triggered);  // Wait for the generator to signal that it is done
    $display("---------------------------------------------");
    $display("Error Count :%0d", sco.err);  // Display the error count from the scoreboard
    $display("---------------------------------------------");
    $finish();                 // Finish the simulation
  endtask
  
  // Run task to execute the pre-test, test, and post-test tasks in sequence
  task run();
    pre_test();               // Call the pre-test task
    test();                   // Call the test task
    post_test();              // Call the post-test task
  endtask
  
endclass

///////////////////////////////////////////////////////

// Testbench module to instantiate and connect the DUT with the environment
module tb;
    
  fifo_if fif();  // Instantiate the FIFO interface
  FIFO dut (fif.clock, fif.rst, fif.wr, fif.rd, fif.data_in, fif.data_out, fif.empty, fif.full);
  // Instantiate the FIFO DUT and connect it to the interface signals
    
  // Initial block to initialize the clock signal
  initial begin
    fif.clock <= 0;  // Initialize the clock to 0
  end
    
  // Generate a clock signal with a period of 20 time units
  always #10 fif.clock <= ~fif.clock;
    
  environment env;  // Declare an instance of the environment class
    
  // Initial block to create and run the testbench environment
  initial begin
    env = new(fif);       // Instantiate the environment with the FIFO interface
    env.gen.count = 20;   // Set the number of transactions to generate
    env.run();            // Run the test environment
  end
    
  // Initial block to set up waveform dumping for simulation analysis
  initial begin
    $dumpfile("dump.vcd");  // Specify the VCD file for waveform dumping
    $dumpvars;              // Dump all variables for waveform analysis
  end
   
endmodule

 
