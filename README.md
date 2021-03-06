# README
MAP-IT is a graph refinement algorithm for inferring the IP addresses used for inter-AS links from traceroute.
It can be used with existing datasets and is intended to work for networks with and without traceroute vantage points.
This is the code described in the paper "[MAP-IT: Multipass Accurate Passive Inferences from Traceroute](http://www.seas.upenn.edu/~amarder/aslinks.html)" [Marder and Smith IMC '16].
It was slightly modified to improve usability.

## REQUIREMENTS
The code was developed using python3.4.
It and later versions of python are supported.
There is currently no python2.* support.

## Python packages:
- py-radix (pip)
- pandas (conda, pip)
- numpy (conda, pip)

## Other:
- [scamper](https://www.caida.org/tools/measurement/scamper/) by Matthew Luckie (for sc_warts2json) if working with traceroute files
- gzip/bzip2 if using compressed files

# INSTRUCTIONS
Until I create a manpage, instructions for running the code will be here.

The primary file is mapit.py.
It can be run using the command python3 mapit.

There are several options, some combination of which is required.

### Creating interface-level graph
- Using the -t <regex> option will tell mapit to create the interface-level graph from the traceroutes
- The argument to the -t option is a Unix-style filename regex
- <regex> should be placed in quotes to avoid premature Unix expansion
- Currently mapit supports a single regex, but any regex which Unix accepts will work (which allows for arbitrary ORs)
- Using the --trace-exit <filename> option will cause mapit to derive the adjacencies from the traces, print the adjacencies to the specified file, and exit (use - for stdout)
- To use a precomputed set of adjacencies, use the -a <filename> option
- mapit processes the traceroute files sequentially. On many machines it is possible for significant speed ups by processing the files in parallel. It is future work to support this in a general manner
- Only warts, warts.gz, and warts.bzip2 are supported. To use other formats, process separately and supply a file with the adjacencies.

### Set of seen addresses
- If the -t option is supplied, then mapit will create a set of seen addresses from the traceroutes
- A set of addresses can also be supplied using the -c <filename> option
- If both the -t and -c options are supplied, the union of the generated and supplied sets of addresses will be used
- The set of addresses is used for the heuristic which attempts to identify the address used for the other side of the point-to-point link
- Using the --addresses-exit <filename> option will write the set of addresses seen in the traceroutes to the specified file (use - for stdout). This set might be different than the set seen in the adjacencies list if traces are discarded due to cycles.
- See the -i option for an alternative

### IP2AS mappings
- The -b <filename> option is used to supply a file of BGP prefix to AS mappings
- The files must be in the [format used by CAIDA](http://data.caida.org/datasets/routing/routeviews-prefix2as/README.txt) for their pfx2AS files (NetworkAddress, Prefixlen, AS)
- The -x <filename> option is used to specify a file containing a list of ASNs used at IXPs. The prefixes that map to these ASes will be treated as IXP prefixes.
- The -y <filename> option is used to specify a file containing a list of IXP prefixes
- See the -i option for an alternative (as well as for use with IPv6)

### AS2ORG mappings
- To help overcome the challenges caused by sibling ASes, mapit really uses the organization that an AS belongs to when identifying inter-AS links (more accurately inter-Org links)
- A list of AS2ORG mappings, in the [format used by CAIDA](http://data.caida.org/datasets/as-organizations/README.txt), can be supplied using the -o <filename> option
- A good place to start is to use CAIDA's AS2ORG mappings based primarily on whois information
- If the -o option is not used, the ASes will be treated as separate ORGs
- See the -i option for an alternative

### Interface Information (-i option)
- The -i <filename> option can be used instead of supplying the set of addresses, IP2AS mappings, and AS2ORG mappings
- If the -i option is used, the -bcoxy options are all ignored
- The specified file must be a CSV with the following headers:
  * Address
  * ASN - the IP2AS mapping of the address
  * Org - the AS2ORG mapping of the address' ASN (can be the same as the ASN but cannot be empty)
  * Otherside - the other side of the address' link
- If using with IPv6, this is currently required. There is no heuristic included for IPv6 other side addresses.
- Using the --interface-exit <filename> option will print such a CSV, derived from the information provided, to the specified file (use - for stdout), and exit

### Factor
- The -f <float> option can be used to further restrict the inferences mapit draws from the dataset
- The default is 0, which means that if any ORG appears more than any other adjacent to the interface, and the ORG is different from the interface's ORG, an inter-AS link will be inferred
- Higher values require the ORG to account for a greater fraction of the ORGs seen adjacent to the interface. As an example, -f 0.5 requires that the ORG account for at least half of the ORGs seen adjacent to the interface. -f 1 requires that all ORGs seen adjacent to the interface are the same.
- For more information, please see the paper

### Providers
- The -p <filename> option specifies a file with a list of ISP ASNs (will be converted to ints)
- The -q <filename> option specified a file with a list of ISP ORGs (will not be converted to ints)
- The -p and -q options are mutually exclusive and should not be used at the same time
- This is used with the ISP->Stub heuristic
- It is not required to specify a list of ISPs, but if neither option is supplied then the heuristic will not be used, which will likely reduce coverage

### Results
- The -w <filename> option can be used to specify the output file for the CSV containing the inter-AS link interfaces (use - for stdout)
- If -w is not used, the results will print to stdout
- The results have the following columns:
  * Address
  * ASN - IP2AS mapping for the address
  * ORG - AS2ORG mapping for the address' ASN
  * Otherside - the interface address used for the other side of the link
  * ConnASN - The ASN connected by the link
  * ConnORG - The ORG connected by the link
  * Direction - True indicates the interface is on a router operated by the connected network. False indicates it's on a router controlled by its network.