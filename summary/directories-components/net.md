# net

Main file is **socket.c**

Families are independant socket types

```c
static const char * const pf_family_names[] = {
	[PF_UNSPEC]	= "PF_UNSPEC",
	[PF_UNIX]	= "PF_UNIX/PF_LOCAL",
	[PF_INET]	= "PF_INET",
	[PF_AX25]	= "PF_AX25",
	[PF_IPX]	= "PF_IPX",
	[PF_APPLETALK]	= "PF_APPLETALK",
	[PF_NETROM]	= "PF_NETROM",
	[PF_BRIDGE]	= "PF_BRIDGE",
	[PF_ATMPVC]	= "PF_ATMPVC",
	[PF_X25]	= "PF_X25",
	[PF_INET6]	= "PF_INET6",
	[PF_ROSE]	= "PF_ROSE",
	[PF_DECnet]	= "PF_DECnet",
	[PF_NETBEUI]	= "PF_NETBEUI",
	[PF_SECURITY]	= "PF_SECURITY",
	[PF_KEY]	= "PF_KEY",
	[PF_NETLINK]	= "PF_NETLINK/PF_ROUTE",
	[PF_PACKET]	= "PF_PACKET",
	[PF_ASH]	= "PF_ASH",
	[PF_ECONET]	= "PF_ECONET",
	[PF_ATMSVC]	= "PF_ATMSVC",
	[PF_RDS]	= "PF_RDS",
	[PF_SNA]	= "PF_SNA",
	[PF_IRDA]	= "PF_IRDA",
	[PF_PPPOX]	= "PF_PPPOX",
	[PF_WANPIPE]	= "PF_WANPIPE",
	[PF_LLC]	= "PF_LLC",
	[PF_IB]		= "PF_IB",
	[PF_MPLS]	= "PF_MPLS",
	[PF_CAN]	= "PF_CAN",
	[PF_TIPC]	= "PF_TIPC",
	[PF_BLUETOOTH]	= "PF_BLUETOOTH",
	[PF_IUCV]	= "PF_IUCV",
	[PF_RXRPC]	= "PF_RXRPC",
	[PF_ISDN]	= "PF_ISDN",
	[PF_PHONET]	= "PF_PHONET",
	[PF_IEEE802154]	= "PF_IEEE802154",
	[PF_CAIF]	= "PF_CAIF",
	[PF_ALG]	= "PF_ALG",
	[PF_NFC]	= "PF_NFC",
	[PF_VSOCK]	= "PF_VSOCK",
	[PF_KCM]	= "PF_KCM",
	[PF_QIPCRTR]	= "PF_QIPCRTR",
	[PF_SMC]	= "PF_SMC",
	[PF_XDP]	= "PF_XDP",
	[PF_MCTP]	= "PF_MCTP",
};
```

![](<../../.gitbook/assets/image (1).png>)
