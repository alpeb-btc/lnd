# `lntest`

`lntest` is a package which holds the components used for the `lnd`’s
integration tests. It is responsible for managing `lnd` nodes, chain backends
and miners, advancing nodes’ states and providing assertions.

### Quick Start

A simple example to run the integration test.

```go
func TestFoo(t *testing.T) {
	// Get the binary path and setup the harness test.
	//
	// TODO: define the binary path to lnd and the name of the database
	// backend.
	harnessTest := lntemp.SetupHarness(t, binary, *dbBackendFlag)
	defer harnessTest.Stop()

	// Setup standby nodes, Alice and Bob, which will be alive and shared
	// among all the test cases.
	harnessTest.SetupStandbyNodes()

	// Run the subset of the test cases selected in this tranche.
	//
	// TODO: define your own testCases.
	for _, tc := range testCases {
		tc := tc

		t.Run(tc.Name, func(st *testing.T) {
			// Create a separate harness test for the testcase to
			// avoid overwriting the external harness test that is
			// tied to the parent test.
			ht, cleanup := harnessTest.Subtest(st)
			defer cleanup()

			// Run the test cases.
			ht.RunTestCase(tc)
		})
	}
}
```

### Package Structure

This package has four major components, `HarnessTest`, `HarnessMiner`,
`node.HarnessNode` and `rpc.HarnessRPC`, with the following architecture,

```
+----------------------------------------------------------+
|                                                          |
|                        HarnessTest                       |
|                                                          |
| +----------------+  +----------------+  +--------------+ |
| |   HarnessNode  |  |   HarnessNode  |  | HarnessMiner | |
| |                |  |                |  +--------------+ |
| | +------------+ |  | +------------+ |                   |
| | | HarnessRPC | |  | | HarnessRPC | |  +--------------+ |
| | +------------+ |  | +------------+ |  | HarnessMiner | |
| +----------------+  +----------------+  +--------------+ |
+----------------------------------------------------------+
```

- `HarnessRPC` holds all the RPC clients and adds a layer over all the RPC
  methods to assert no error happened at the RPC level.

- `HarnessNode` builds on top of the `HarnessRPC`. It is responsible for
  managing the `lnd` node, including start and stop pf the `lnd` process,
  authentication of the gRPC connection, topology subscription(`NodeWatcher`)
  and maintains an internal state(`NodeState`).

- `HarnessMiner` builds on top of `btcd`’s `rcptest.Harness` and is responsilbe
  for managing blocks and the mempool.

- `HarnessTest` builds on top of `testing.T` and can be viewed as the assertion
  machine. It provides multiple ways to initialize a node, such as with/without
  seed, backups, etc. It also handles interactions between nodes like
  connecting nodes and opening/closing channels so it’s easier to acquire or
  validate a desired test states such as node’s balance, mempool condition,
  etc.

### Standby Nodes

Standby nodes are `HarnessNode`s created when initializing the integration test
and stay alive across all the test cases. Creating a new node is not without a
cost. With block height increasing, it takes significantly longer to initialize
a new node and wait for it to be synced. Standby nodes, however, don’t have
this problem as they are digesting blocks all the time. Thus it’s encouraged to
use standby nodes wherever possible.

Currently there are two standby nodes, Alice and Bob. Their internal states are
recorded and taken into account when `HarnessTest` makes assertions. When
making a new test case using `Subtest`, there’s a cleanup function which
further validates the current test case has no dangling uncleaned states, such
as transactions left in mempool, open channels, etc.

