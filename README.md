# EC513 Spring 2023 Project
Clone the Gem5 repository and implement the policy of your choosing. Please follow the guidelines in the project document.

Run the simulations according to the project document and upload your source code and the results in the repo.
Your results should be in the `output/` directory.

## File Structure Explanation
The file structure includes the entirety of the gem5 folder that was worked in, a summary of files changed within the files_changed directory, and an output folder than contains an excel sheet with all the compiled data from our ran tests.

### files_changed directory
This directory includes all the files that were created and changed to properly implement our cache replacement policy (LFUDA). Specifically, we created/edited the following files: lfuda_rp.hh, lfuda_rp.cc, ReplacementPolicies.py, SConscript, and RubyCache.py. 

#### lfuda_rp.hh and lfuda_rp.cc
The files are the header and C++ implementation for our cache replacement policy: LFUDA. These files are based on the base LFU files but with some slight modifications. To highlight these changes, we first added two variables, both unsigned integers called “agingValue” and “key” (both initialized to zero). “agingValue” is the variable that holds the current refCount (frequency) of the most recently evicted cache block. The “key” value is effectively our evaluation criteria for cache block blocks with a lower “key” value will be evicted
The relevant function changes occur in touch(), reset(), and getVictim(). 

In touch(), we increment refCount by 1 just like in LFU. After incrementing refCount we perform the following operation: key = refCount + agingValue. Doing so allows new cache blocks the ability to compete against older, stale ones. 
``` cpp
void
LFUDA::touch(const std::shared_ptr<ReplacementData>& replacement_data) const
{
    // Update reference count
    auto data = std::static_pointer_cast<LFUDAReplData>(replacement_data);
    data->refCount++;
    data->key = data->refCount + agingValue;
}
```

In reset(), since this is where we initialize the cache block, we set refCount = agingValue for the same reason as before. We want newer cache blocks to have the ability to compete with older, stale ones.
``` cpp
void
LFUDA::reset(const std::shared_ptr<ReplacementData>& replacement_data) const
{
    // Reset reference count
    std::static_pointer_cast<LFUDAReplData>(replacement_data)->refCount = agingValue + 1;
}

```

Finally, in getVictim() we notice small changes to the algorithm except that eviction is now based on the “key” value rather than refCount. After determining the victim for eviction, we set “agingValue” to the refCount of the victim.
``` cpp
ReplaceableEntry*
LFUDA::getVictim(const ReplacementCandidates& candidates) const
{
    // There must be at least one replacement candidate
    assert(candidates.size() > 0);

    // Visit all candidates to find victim
    ReplaceableEntry* victim = candidates[0];
    for (const auto& candidate : candidates) {
        // Update victim entry if necessary
        if (std::static_pointer_cast<LFUDAReplData>(
                    candidate->replacementData)->key <
                std::static_pointer_cast<LFUDAReplData>(
                    victim->replacementData)->key) {
            victim = candidate;
        }
    }

    agingValue = std::static_pointer_cast<LFUDAReplData>(victim->replacementData)->refCount;

    return victim;
}
```

#### ReplacementPolicies.py
For ReplacementPolicies.py we simply had to add a class declaration called LFUDARP with inheritance from BaseReplacementPolicy. This class specifies the type, class name, and relevant directory where the header file exists. The code can be seen below:

``` python
class LFUDARP(BaseReplacementPolicy):
    type = "LFUDARP"
    cxx_class = "gem5::replacement_policy::LFUDA"
    cxx_header = "mem/cache/replacement_policies/lfuda_rp.hh"
```

#### SConscript
For SConscript we simply had to add our LFUDARP() that was defined in ReplacementPolicies.py to the list of available sim_objects such that we can simulate our new replacement policy. The change looks like the following: 

``` python
SimObject('ReplacementPolicies.py', sim_objects=[
    'BaseReplacementPolicy', 'DuelingRP', 'FIFORP', 'SecondChanceRP',
    'LFURP', 'LRURP','LFUDARP', 'BIPRP', 'MRURP', 'RandomRP', 'BRRIPRP', 'SHiPRP',
    'SHiPMemRP', 'SHiPPCRP', 'TreePLRURP', 'WeightedLRURP'])
```

Furthermore, we add our header and C++ files as source files with the following line:
``` python
Source('lfuda_rp.cc')
```

#### RubyCache.py
For this file all we need to change is to specify the cache replacement policy that we are using. Since we are testing our implemented cache replacement policy, we enter LFUDARP(), as created and specified in our ReplacementPolicies.py. The line looks like the following: 

``` python
Param.BaseReplacementPolicy(LFUDARP(), "")
```

#### project.sh
This was a bash script that was created to help us automate collecting data for a specific benchmark while automatically saving it on completion. It did so by first sourcing the relevant functions. Then it would run the benchmark using the build/X85/gem.opt command. Finally, it would copy the data from the m5out folder to an appropriate destination. In the case below it saved it to a directory called "oneMB" for the mcf_r benchmark. 

``` bash
source /projectnb/ec513/materials/HW2/spec-2017/sourceme

# run benchmark
build/X86/gem5.opt \
configs/example/gem5_library/x86-spec-cpu2017-benchmarks.py \
--image ../disk-image/spec-2017/spec-2017-image/spec-2017 \
--partition 1 \
--benchmark 505.mcf_r \
--size test

# save benchmark results
mkdir -p /projectnb/ec513/students/cris0211/HW2/spec-2017/gem5/oneMB
cp -a /projectnb/ec513/students/cris0211/HW2/spec-2017/gem5/m5out /projectnb/ec513/students/cris0211/HW2/spec-2017/gem5/oneMB

```

### gem5 directory
This is a very large folder with many contents thus an explanation for how to find all the relevant changed files will be included below.

1. lfuda_rp.hh and lfuda_rp.cc: Can be found in gem5/src/mem/cache/replacement_policies

2. ReplacementPolicies.py: Can be found in gem5/src/mem/cache/replacement_policies

3. SConscript: Can be found in gem5/src/mem/cache/replacement_policies

4. RubyCache.py: Can be found gem5/src/mem/ruby/structures

Other relevant file(s):

1. project.sh (script used to automate collecting and saving data): gem5/
