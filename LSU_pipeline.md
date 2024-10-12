Pipeline of the multi-pipeline LSU unit will have the following stages:

* Dispatch:
   * Allocate entry in ldst_inst_queue_ for any load/store instruction

* Issue (up to N instructions per cycle):
  * Perform Store Set Prediction (still thinking of implementation):
    * For each load/store instruction, consult the Store Set Predictor
    * Predict potential dependencies with older stores
    * Use predictions to guide initial scheduling decisions
  * If stall_pipe_on_miss is false, prioritize issuing from replay buffer:
    * Instructions remain in replay buffer until explicitly removed
    * Dropped instructions become re-issuable after N cycles (configurable)
    * Instructions may undergo multiple replay attempts
  * If replay buffer empty or not prioritized, issue from ready_queue_:
    * Loads are ready when:
      * If allow_spec_load_exec is false: dependencies met and older store addresses known
      * If allow_spec_load_exec is true: dependencies met, regardless of older store status
      * Consider Store Set Prediction results for more accurate readiness determination (not sure how to implement yet)
    * Stores are ready when both data and address operands are available
  * For each issued instruction:
    * If it's a load:
      * Allocate to an available ldst_pipeline_
    * If it's a store:
      * Allocate to both an available ldst_pipeline_ and an available store_data_pipeline_
    * If stall_pipe_on_miss is false, add to replay buffer
    * Record Store Set Prediction information for later verification
  * Update issue slot availability for next cycle

* Address Generation and Initial Dependency Check (in each ldst_pipeline_):
  * Generate virtual address
  * For stores, associate generated address with corresponding store_data_pipeline_ entry
  * Update dependency tracking with newly generated address
  * Use address predictions and partial information to start conflict checks
  * Preliminarily check against
    * Predicted conflicts from issue stage
    * Approximate address ranges of in-flight memory operations

* Refined Dependency Check & Speculation:
   * Combine results from Address Generation and Initial Dependency Check
   * Update dependency tracking with newly generated address
   * Verify predictions made during issue stage
   * Check for conflicts against:
      * Committed Store Buffer (CSB)
      * In-flight stores with resolved addresses
      * In-flight loads (for stores)
   * If conflict detected:
      * For loads: attempt data forwarding, otherwise mark for replay
      * For stores: identify affected younger loads for potential replay
   * Update Store Sets Predictor or Memory Dependency Predictor
   * Handle bank conflicts:
      * Allow one access to proceed
      * Mark others for replay
    
* Store Data Pipeline (parallel to address pipeline, in each store_data_pipeline_):
  * Data Preparation:
    * Generate or retrieve store data
  * Data Ready:
    * Signal readiness to address pipeline
    * Coordinate with address pipeline for completion

* MMU Lookup:
  * If hit:
    * Obtain physical address
    * Update LoadQ/StoreQ with physical address
    * For loads:
      * Check StoreQ for address matches
      * If match found, prepare for data forwarding
    * For stores (if allow_spec_load_exec is true):
      * Check LoadQ for younger load address matches
      * If match found, mark younger load and its dependents for replay
   * If miss or MMU busy:
    * If stall_pipe_on_miss is true:
        * Stall pipeline until miss resolved
    * If stall_pipe_on_miss is false:
        * Drop instruction from pipeline
        * Mark for replay after backoff period
   * On miss, initiate page table walk

* Cache Lookup:
   * For loads:
      * If store forwarding identified, bypass cache lookup
      * If cache busy, drop instruction and mark for replay
      * On hit: proceed to next stage
      * On miss:
         * If stall_pipe_on_miss is true: stall until miss resolved
         * If stall_pipe_on_miss is false:
            * Drop from pipeline and mark for replay
            * Check MSHR for existing entry
            * If no matching MSHR and not full, allocate new MSHR and line fill buffer
   * For stores:
      * Perform lookup to determine cache line status
      * On miss, handle similarly to load miss

* Cache Data Read (loads only):
   * Read data from cache or forwarding path
   * Prepare for writeback
   * Remove load from replay buffer if hit

* Complete:
   * For loads:
      * Write data to register file
      * Signal dependent instructions
      * Mark as completed in the ROB
   * For stores:
      * Coordinate with data pipeline
      * Mark as complted in th ROB, but not yet committed
   * Update Store Buffer for stores
   * Remove instruction from replay buffer
   * Deallocate from ldst_inst_queue_

* Retire
  * retire should be handled in ROB
