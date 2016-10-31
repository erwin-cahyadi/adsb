package main

import (
	"net"
	"strconv"
	"fmt"
//	"bufio"
//	"bytes"
	"strings"
	"math"
	"os"
//	"encoding/hex"
	"github.com/vaughan0/go-ini"
	"time"
)

var mm modesMessage
var udp [10]net.Conn
var str string

const c21def = 0
const c21len0 = 1
const c21len = 2
const c21fsd1 = 3
const c21fsd2 = 4
const c21fsd3 = 5
const c21fsd4 = 6
const c21fsd5 = 7

const ais_charset = "?ABCDEFGHIJKLMNOPQRSTUVWXYZ????? ???????????????0123456789??????"
const MODES_LONG_MSG_BITS = 112
const MODES_SHORT_MSG_BITS = 56
const AIRCRAFT_TTL = 180000

var modes_checksum_table = [112]uint32{
0x3935ea, 0x1c9af5, 0xf1b77e, 0x78dbbf, 0xc397db, 0x9e31e9, 0xb0e2f0, 0x587178,
0x2c38bc, 0x161c5e, 0x0b0e2f, 0xfa7d13, 0x82c48d, 0xbe9842, 0x5f4c21, 0xd05c14,
0x682e0a, 0x341705, 0xe5f186, 0x72f8c3, 0xc68665, 0x9cb936, 0x4e5c9b, 0xd8d449,
0x939020, 0x49c810, 0x24e408, 0x127204, 0x093902, 0x049c81, 0xfdb444, 0x7eda22,
0x3f6d11, 0xe04c8c, 0x702646, 0x381323, 0xe3f395, 0x8e03ce, 0x4701e7, 0xdc7af7,
0x91c77f, 0xb719bb, 0xa476d9, 0xadc168, 0x56e0b4, 0x2b705a, 0x15b82d, 0xf52612,
0x7a9309, 0xc2b380, 0x6159c0, 0x30ace0, 0x185670, 0x0c2b38, 0x06159c, 0x030ace,
0x018567, 0xff38b7, 0x80665f, 0xbfc92b, 0xa01e91, 0xaff54c, 0x57faa6, 0x2bfd53,
0xea04ad, 0x8af852, 0x457c29, 0xdd4410, 0x6ea208, 0x375104, 0x1ba882, 0x0dd441,
0xf91024, 0x7c8812, 0x3e4409, 0xe0d800, 0x706c00, 0x383600, 0x1c1b00, 0x0e0d80,
0x0706c0, 0x038360, 0x01c1b0, 0x00e0d8, 0x00706c, 0x003836, 0x001c1b, 0xfff409,
0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000,
0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000,
0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000, 0x000000}


type Aircraft struct {
	addr uint32
	flight string
	altitude int
	vrate int
	speed int
	track int
	odd_cprlat int
	odd_cprlon int
	even_cprlat int
	even_cprlon int
	lat float64
	lon float64
	odd_cprtime uint64
	even_cprtime uint64
	seen time.Time
	next *Aircraft
}

type Node struct {
	Aircraft
	next, prev	*Node
}

type List struct {
	head, tail *Node
}

func (l *List) First() *Node {
	return l.head
}

func (n * Node) Next() *Node {
	return n.next
}

func (n * Node) Prev() *Node {
	return n.prev
}

func (l *List) Push(a Aircraft) *List {
	n := &Node{Aircraft: a}
	if l.head == nil { /* First Node */
		l.head = n
	} else {
		l.tail.next = n /* Add after prev last node */
		n.prev = l.tail /* Link back to prev last node */
	}
	l.tail = n			/* reset tail to newly added node*/
	return l
}

func (l * List) Find(addr uint32) *Node {
	found := false
	var ret *Node = nil
	for n := l.First(); n != nil && !found; n = n.Next() {
		if n.Aircraft.addr == addr {
			found = true
			ret = n
		}	
	}
	return ret
}

func (l * List) Delete(addr uint32) bool {
	success := false
	found := false
	var node2del *Node

	for n := l.First(); n != nil && !found; n = n.Next() {
		if n.Aircraft.addr == addr {
			found = true
			node2del = n
		}
	}
	if node2del != nil {
		if node2del.prev == nil && node2del.next == nil {
			/* Last node */
			l.head = nil
			l.tail = nil
			success = true
		} else if node2del.prev == nil {
			/* First node */
			l.head = node2del.next
			l.head.prev = nil
			success = true
		} else if node2del.next == nil {
			/* Last node */
			l.tail = node2del.prev
			l.tail.next = nil
			success = true
		} else {
			prev_node := node2del.prev
			next_node := node2del.next
			/* remove this node */
			prev_node.next = node2del.next
			next_node.prev = node2del.prev
			success = true
		}
	}
	return success
}

func displayAircrafts(l *List) {
	var a Aircraft
	fmt.Print("\033[H\033[2J")
	fmt.Print(" ICAO   Flight   Altitude  Vrate  Speed   Track   Latitude   Longitude   TTL\n")
	fmt.Print("=============================================================================\n")
	for n := l.First(); n != nil; n = n.Next() {
		a = n.Aircraft
		stale_time := time.Since(n.Aircraft.seen)
		ttl := AIRCRAFT_TTL - stale_time.Nanoseconds() / 1000000
		fmt.Printf("%06X  %8s  %5d   %5d  %5d   %5d   %3.6f   %3.6f  %d\n", a.addr, a.flight, a.altitude, a.vrate, a.speed, a.track, a.lat, a.lon, ttl / 1000)
	}
	fmt.Printf("\n")	
}

type modesMessage struct {
	msg []byte

	msgbits int
	crc uint32

	length int
	nanosec uint64
	daysec uint64
	time uint64
	hour,min,sec,sec1000 uint64

	msgtype int		/* Downlink Format # */
	icao1 int		/* ICAO Address byte 1 */
	icao2 int		/* ICAO Address byte 2 */
	icao3 int		/* ICAO Address byte 3 */

	/* DF11 */
	ca int			/* Responder capabilities */

	/* DF17 */
	metype int				/* Extended squitter message type */
	mesub int				/* Extended squitter message subtype */
	heading_is_valid int
	heading float64
	aircraft_type int
	fflag int				/* 1 = Odd, 0 = Even CPR message */
	tflag int				/* UTC synchronized? */
	raw_latitude int		/* Non decoded latitude */
	raw_longitude int		/* Non decoded longitude */
	cpr_decoded int
	latitude float64
	longitude float64
	identification [9]byte
	flight [9]uint8			/* 8 chars flight number */
	ew_dir int				/* 0 = East, 1 = West */
	ew_velocity int			/* E/W velocity */
	ns_dir int				/* 0 = North, 1 = South */
	ns_velocity int			/* N/S velocity */
	vert_rate_source int	/* Vertical rate source */
	vert_rate_sign	int 	/* Vertical rate sign	*/
	vert_rate int			/* Vertical rate */
	velocity float64			/* Computed from EW and NS velocity */
	airspeed int
	airspeed_type int
	sbnic int
	NIC	int
	PIC int
	NAC int

	/* DF4, DF5, DF20, DF21 */
	fs int				/* Flight status */
	dr int				/* Request extraction of downlink request */
	um int				/* Request extraction of downlink request */
	identity int		/* 13 bits identity (Squawk) */
	squawk int

	/* Field used by multiple message type */
	altitude int
	unit int
}

func modesMessageLenByType(msgtype int) int{
	if (msgtype == 16 || msgtype == 17 || msgtype == 19 || msgtype == 20 || msgtype == 21) {
		return MODES_LONG_MSG_BITS
	} else {
		return MODES_SHORT_MSG_BITS
	}
}

func modesChecksum(msg []byte, bits int) uint32{
	var crc uint32
	var offset int
	crc = 0
	if(bits == 112) {
		offset = 0
	} else {
		offset = 56
	}
	for j:= 0; j < bits; j++ {
		by := j / 8
		bit := j % 8
		bitmask := (byte)(1 << (uint)(7 - bit))
		if(msg[by] & bitmask != 0) {
			crc ^= modes_checksum_table[j+offset]
		}
	}
	return crc
}

func cprModFunction(a int, b int) int{
	res := a % b
	if (res < 0) {res += b}
	return res
}

func cprNLFunction(lat float64) int {
	if (lat < 0) {lat = -lat} /* Table is simmetric about the equator. */
	if (lat < 10.47047130) { return 59 }
	if (lat < 14.82817437) { return 58 }
	if (lat < 18.18626357) { return 57 }
	if (lat < 21.02939493) { return 56 }
	if (lat < 23.54504487) { return 55 }
	if (lat < 25.82924707) { return 54 }
	if (lat < 27.93898710) { return 53 }
	if (lat < 29.91135686) { return 52 }
	if (lat < 31.77209708) { return 51 }
	if (lat < 33.53993436) { return 50 }
	if (lat < 35.22899598) { return 49 }
	if (lat < 36.85025108) { return 48 }
	if (lat < 38.41241892) { return 47 }
	if (lat < 39.92256684) { return 46 }
	if (lat < 41.38651832) { return 45 }
	if (lat < 42.80914012) { return 44 }
	if (lat < 44.19454951) { return 43 }
	if (lat < 45.54626723) { return 42 }
	if (lat < 46.86733252) { return 41 }
	if (lat < 48.16039128) { return 40 }
	if (lat < 49.42776439) { return 39 }
	if (lat < 50.67150166) { return 38 }
	if (lat < 51.89342469) { return 37 }
	if (lat < 53.09516153) { return 36 }
	if (lat < 54.27817472) { return 35 }
	if (lat < 55.44378444) { return 34 }
	if (lat < 56.59318756) { return 33 }
	if (lat < 57.72747354) { return 32 }
	if (lat < 58.84763776) { return 31 }
	if (lat < 59.95459277) { return 30 }
	if (lat < 61.04917774) { return 29 }
	if (lat < 62.13216659) { return 28 }
	if (lat < 63.20427479) { return 27 }
	if (lat < 64.26616523) { return 26 }
	if (lat < 65.31845310) { return 25 }
	if (lat < 66.36171008) { return 24 }
	if (lat < 67.39646774) { return 23 }
	if (lat < 68.42322022) { return 22 }
	if (lat < 69.44242631) { return 21 }
	if (lat < 70.45451075) { return 20 }
	if (lat < 71.45986473) { return 19 }
	if (lat < 72.45884545) { return 18 }
	if (lat < 73.45177442) { return 17 }
	if (lat < 74.43893416) { return 16 }
	if (lat < 75.42056257) { return 15 }
	if (lat < 76.39684391) { return 14 }
	if (lat < 77.36789461) { return 13 }
	if (lat < 78.33374083) { return 12 }
	if (lat < 79.29428225) { return 11 }
	if (lat < 80.24923213) { return 10 }
	if (lat < 81.19801349) { return 9 }
	if (lat < 82.13956981) { return 8 }
	if (lat < 83.07199445) { return 7 }
	if (lat < 83.99173563) { return 6 }
	if (lat < 84.89166191) { return 5 }
	if (lat < 85.75541621) { return 4 }
	if (lat < 86.53536998) { return 3 }
	if (lat < 87.00000000) { return 2 }
    return 1
}

func cprNFunction(lat float64, isodd int) int {
	nl := cprNLFunction(lat) - isodd
	if(nl < 1) {nl = 1}
	return nl
}

func cprDlonFunction(lat float64, isodd int) float64 {
	return 360.0 / (float64)(cprNFunction(lat, isodd))
}

func decodeCPR(a *Aircraft) {
	const AirDlat0 float64 = 360.0 / 60
	const AirDlat1 float64 = 360.0 / 59
	lat0 := (float64)(a.even_cprlat) 
	lat1 := (float64)(a.odd_cprlat) 
	lon0 := (float64)(a.even_cprlon) 
	lon1 := (float64)(a.odd_cprlon) 

	// Compute the latitude index "j"
	j := (int)(math.Floor(((59*lat0 - 60*lat1) / 131072) + 0.5))
	rlat0 := AirDlat0 * ((float64)(cprModFunction(j,60)) + lat0 / 131072)
	rlat1 := AirDlat1 * ((float64)(cprModFunction(j,59)) + lat1 / 131072)

	if(rlat0 >= 270) {rlat0 -= 360}
	if(rlat1 >= 270) {rlat1 -= 360}

	// Check that both are in the same latitude zone, or abort
	if(cprNLFunction(rlat0) != cprNLFunction(rlat1)) { return }
	if(a.even_cprtime > a.odd_cprtime) {
		// Use even packet
		ni := cprNFunction(rlat0, 0)
		m := (int)(math.Floor((((lon0 * (float64)(cprNLFunction(rlat0)-1)) - (lon1 * (float64)(cprNLFunction(rlat0)))) / 131072) + 0.5))
        a.lon = cprDlonFunction(rlat0,0) * ((float64)(cprModFunction(m,ni))+lon0/131072);
        a.lat = rlat0;		
	} else {
		// Use odd packet
		ni := cprNFunction(rlat1, 1)
		m := (int)(math.Floor((((lon0 * (float64)(cprNLFunction(rlat1)-1)) - (lon1 * (float64)(cprNLFunction(rlat1)))) / 131072) + 0.5))
        a.lon = cprDlonFunction(rlat1,0) * ((float64)(cprModFunction(m,ni))+lon1/131072);
        a.lat = rlat1;		
	}
	if (a.lon > 180) {a.lon -= 360}
}

func decodeNIC(tc int, sbnic int) (int, int) {
	var NIC, PIC int
	switch tc {
	case 9:
		NIC = 11
		PIC = 14
	case 10:
		NIC = 10
		PIC = 13
	case 11:
		if (sbnic == 1) {
			NIC = 9
			PIC = 12
		} else {
			NIC = 8
			PIC = 11
		}
	case 12:
		NIC = 7
		PIC = 10
	case 13:
		NIC = 6
		if (sbnic == 1) {
			PIC = 9
		} else {
			PIC = 8
		}
	case 14:
		NIC = 5
		PIC = 6
	case 15:
		NIC = 4
		PIC = 5
	case 16:
		if (sbnic == 1) {
			NIC = 3
			PIC = 4
		} else {
			NIC = 2
			PIC = 3
		}
	case 17:
		NIC = 1
	case 18:
		NIC = 0
	}
	return NIC, PIC
}

func displayModesMessage(mm modesMessage) {
	ac_type_str := []string{"Aircraft Type D", "Aircraft Type C", "Aircraft Type B", "Aircraft Type A"}
	fs_str := []string{"Normal, Airborne", "Normal, On the ground", "ALERT, Airborne", "ALERT, On the ground",
						"ALERT & Special Position Identification. Airborne or Ground",
						"Special Position Identification. Airborne or Ground",
						"Value 6 is not assigned", "Value 7 is not assigned"}
	ca_str := []string{"Level 1 (Surveillance Only)", "Level 2 (DF0,4,5,11)", "Level 3 (DF0,4,5,11,20,21)", "Level 4 (DF0,4,5,11,20,21,24)",
						"Level 2+3+4 (DF0,4,5,11,20,21,24,code 7 - is on ground)",
						"Level 2+3+4 (DF0,4,5,11,20,21,24,code 7 - is on airborne)",
						"Level 2+3+4 (DF0,4,5,11,20,21,24,code 7)",
						"Level 7 ???"}

	fmt.Printf("Message: ")
	for i := 0; i < mm.length; i++ {
		fmt.Printf("%02X ", mm.msg[i])
	}
	fmt.Println()

	fmt.Printf("Time:%02d:%02d:%02d.%03d\n", mm.hour, mm.min, mm.sec, mm.sec1000)
	fmt.Printf("Time High Precision:%02d:%02d:%02d.%09d\n", mm.hour, mm.min, mm.sec, mm.nanosec)
	if(mm.msgtype == 0) {
		/* DF 0 */
		fmt.Printf("DF 0: Short Air-Air Surveillance.\n")
		if (mm.unit == 1) {
			str = "meters"
		} else {
			str = "feet"
		}
		fmt.Printf(" Altitude     : %d %s\n", mm.altitude, str)
		fmt.Printf(" ICAO Address : %02X%02X%02X\n", mm.icao1, mm.icao2, mm.icao3)
	} else if(mm.msgtype == 4 || mm.msgtype == 20) {
		if (mm.msgtype == 4) {
			str = "Surveillance"
		} else {
			str = "Comm-B"
		}
		fmt.Printf("DF %d: %s Altitude Reply.\n", mm.msgtype, str)
		fmt.Printf(" Flight Status : %s\n", fs_str[mm.fs])
		fmt.Printf(" DR            : %d\n", mm.dr)
		fmt.Printf(" UM            : %d\n", mm.um)
		if (mm.unit == 1) {
			str = "meters"
		} else {
			str = "feet"
		}
		fmt.Printf(" Altitude      : %d %s\n", mm.altitude, str)
		fmt.Printf(" ICAO Address  : %02X%02X%02X\n", mm.icao1, mm.icao2, mm.icao3)
				
	} else if(mm.msgtype == 5 || mm.msgtype == 21) {
		if (mm.msgtype == 5) {
			str = "Surveillance"
		} else {
			str = "Comm-B"
		}
		fmt.Printf("DF %d: %s Altitude Reply.\n", mm.msgtype, str)
		fmt.Printf(" Flight Status : %s\n", fs_str[mm.fs])
		fmt.Printf(" DR            : %d\n", mm.dr)
		fmt.Printf(" UM            : %d\n", mm.um)
		fmt.Printf(" Squawk        : %d\n", mm.identity)
		fmt.Printf(" ICAO Address  : %02X%02X%02X\n", mm.icao1, mm.icao2, mm.icao3)
				
	} else if(mm.msgtype == 11) {
		fmt.Printf("DF 11: All Call Reply.\n")
		fmt.Printf(" Capability         : %d %s\n", mm.ca, ca_str[mm.ca])
		fmt.Printf(" ICAO Address       : %02X%02X%02X\n", mm.icao1, mm.icao2, mm.icao3)
				
	} else if(mm.msgtype == 17) {
		/* DF 0 */
		fmt.Printf("DF 17: ADSB Message.\n")
		fmt.Printf(" Capability         : %d %s\n", mm.ca, ca_str[mm.ca])
		fmt.Printf(" ICAO Address       : %02X%02X%02X\n", mm.icao1, mm.icao2, mm.icao3)
		fmt.Printf(" Ext. Squitter Type : %d\n", mm.metype)
		fmt.Printf(" Ext. Squitter Sub  : %d\n", mm.mesub)
		if(mm.metype >= 1 && mm.metype <= 4) {
			/* Aircraft Identification */
			fmt.Printf("    Aircraft Type   : %s\n", ac_type_str[mm.aircraft_type])
			fmt.Printf("    Identification  : %s\n", mm.flight)
		} else if(mm.metype >= 9 && mm.metype <= 18) {
			fmt.Printf("    F flag          : %d\n", mm.fflag)
			fmt.Printf("    T flag          : %d\n", mm.tflag)
			fmt.Printf("    NIC             : %d\n", mm.NIC)
			fmt.Printf("    PIC             : %d\n", mm.PIC)
			fmt.Printf("    Altitude        : %d feet\n", mm.altitude)
			fmt.Printf("    Lat CPR         : %d\n", mm.raw_latitude)
			fmt.Printf("    Lon CPR         : %d\n", mm.raw_longitude)
			if(mm.cpr_decoded == 1) {
				fmt.Printf("    Latitude        : %f\n", mm.latitude)
				fmt.Printf("    Longitude       : %f\n", mm.longitude)
			} else {
				fmt.Printf("    Latitude        : NA\n")
				fmt.Printf("    Longitude       : NA\n")
			}
		} else if(mm.metype >= 19 && mm.mesub >= 1 && mm.mesub <= 4) {
			if(mm.mesub == 1 || mm.mesub == 2) {
				fmt.Printf("    NAC             : %d\n", mm.NAC)
				fmt.Printf("    EW direction    : %d\n", mm.ew_dir)
				fmt.Printf("    EW velocity     : %d knots\n", mm.ew_velocity)
				fmt.Printf("    NS direction    : %d\n", mm.ns_dir)
				fmt.Printf("    NS velocity     : %d knots\n", mm.ns_velocity)
				fmt.Printf("    Velocity        : %f knots\n", mm.velocity)
				fmt.Printf("    Heading         : %f degree\n", mm.heading)
				fmt.Printf("    Vert. rate src  : %d\n", mm.vert_rate_source)
				fmt.Printf("    Vert. rate sign : %d\n", mm.vert_rate_sign)
				fmt.Printf("    Vert. rate      : %d ft/min\n", mm.vert_rate)
			} else if(mm.mesub == 3 || mm.mesub == 4) {
				fmt.Printf("    NAC             : %d\n", mm.NAC)
				fmt.Printf("    Airspeed        : %d knots\n", mm.airspeed)
				if(mm.airspeed_type == 1) {
					fmt.Printf("    Airspeed Type   : True Airspeed\n")
				} else {
					fmt.Printf("    Airspeed Type   : Indicated Airspeed\n")
				}
				fmt.Printf("    Heading status  : %d\n", mm.heading_is_valid)
				fmt.Printf("    Heading         : %f degree\n", mm.heading)
				fmt.Printf("    Vert. rate src  : %d\n", mm.vert_rate_source)
				fmt.Printf("    Vert. rate sign : %d\n", mm.vert_rate_sign)
				fmt.Printf("    Vert. rate      : %d\n ft/min", mm.vert_rate)
			}
		}
	}

	fmt.Println()
}

func decodeCAT21(mm modesMessage, SAC int32, SIC int32) []byte {
	BCat21 := make([]byte, 100)
	BCat21[c21def] = 21 // cat21 = 21
	BCat21[c21len0] = 0 // 
	BCat21[c21len] = 8  // panjang payload awal = 8 = 1header + 2length + 4FSPEC/UAP
	BCat21[c21fsd1] = 1
	BCat21[c21fsd2] = 1
	BCat21[c21fsd3] = 1
	BCat21[c21fsd4] = 1
	BCat21[c21fsd5] = 0
	PPloadC21 := 8 //pointer payload cat21

	if(mm.msgtype == 0) {
		/* Short Air-Air Surveillance */

		// fsd1
		BCat21[c21fsd1] |= 0x80 // Data Source Identifier SAC/SIC
		BCat21[c21len] += 2     // add 2 bytes to total length cat21
		BCat21[PPloadC21] = (byte)(SAC)   // System Area Code
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(SIC)   // System Identification Code
		PPloadC21 += 1          // add pointer

		BCat21[c21fsd1] |= 0x40 // Target Report Descriptor
		BCat21[c21len] += 2     // add 2 bytes to total length cat21
		BCat21[PPloadC21] = 0x01   // 24 bit ICAO Address
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = 0x00   // Actual Target Report
		PPloadC21 += 1          // add pointer

		//fsd2
		BCat21[c21fsd2] |= 0x10 // Target Address
		BCat21[c21len] += 3     // add 3 bytes to total length cat21
		BCat21[PPloadC21] = (byte)(mm.icao1)   // ICAO
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(mm.icao2)   // ICAO
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(mm.icao3)   // ICAO
		PPloadC21 += 1          // add pointer

		//fsd3
		BCat21[c21fsd3] |= 0x20 // Quality Indicator
		BCat21[c21len] += 1     // add 1 byte to total length cat21
		BCat21[PPloadC21] = 0   // Quality
		PPloadC21 += 1          // add pointer
		
		FL := mm.altitude * 4 / 100		
		BCat21[c21fsd3] |= 0x02 // Flight Level
		BCat21[c21len] += 2     // add 1 byte to total length cat21
		BCat21[PPloadC21] = (byte)((FL >> 8) & 0xff)   // Flight Level
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(FL & 0xff)   // Flight Level
		PPloadC21 += 1          // add pointer

		//fsd4
		TOD := (int64)(mm.daysec) * 128 + (int64)(mm.nanosec) / 7812500
		BCat21[c21fsd4] |= 0x02 // Time of report transmission
		BCat21[c21len] += 3     // add 3 bytes to total length cat21
		BCat21[PPloadC21] = (byte)((TOD >> 16) & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)((TOD >> 8) & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(TOD & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer

	} else if(mm.msgtype == 4 || mm.msgtype == 20) {
//		fmt.Printf(" Flight Status : %s\n", fs_str[mm.fs])
//		fmt.Printf(" DR            : %d\n", mm.dr)
//		fmt.Printf(" UM            : %d\n", mm.um)
		// fsd1
		BCat21[c21fsd1] |= 0x80 // Data Source Identifier SAC/SIC
		BCat21[c21len] += 2     // add 2 bytes to total length cat21
		BCat21[PPloadC21] = (byte)(SAC)   // System Area Code
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(SIC)   // System Identification Code
		PPloadC21 += 1          // add pointer

		BCat21[c21fsd1] |= 0x40 // Target Report Descriptor
		BCat21[c21len] += 2     // add 2 bytes to total length cat21
		BCat21[PPloadC21] = 0x01   // 24 bit ICAO Address
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = 0x00   // Actual Target Report
		PPloadC21 += 1          // add pointer

		//fsd2
		BCat21[c21fsd2] |= 0x10 // Target Address
		BCat21[c21len] += 3     // add 3 bytes to total length cat21
		BCat21[PPloadC21] = (byte)(mm.icao1)   // ICAO
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(mm.icao2)   // ICAO
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(mm.icao3)   // ICAO
		PPloadC21 += 1          // add pointer

		//fsd3
		BCat21[c21fsd3] |= 0x20 // Quality Indicator
		BCat21[c21len] += 1     // add 1 byte to total length cat21
		BCat21[PPloadC21] = 0   // Quality
		PPloadC21 += 1          // add pointer
		
		FL := mm.altitude * 4 / 100		
		BCat21[c21fsd3] |= 0x02 // Flight Level
		BCat21[c21len] += 2     // add 1 byte to total length cat21
		BCat21[PPloadC21] = (byte)((FL >> 8) & 0xff)   // Flight Level
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(FL & 0xff)   // Flight Level
		PPloadC21 += 1          // add pointer

		//fsd4
		TOD := (int64)(mm.daysec) * 128 + (int64)(mm.nanosec) / 7812500
		BCat21[c21fsd4] |= 0x02 // Time of report transmission
		BCat21[c21len] += 3     // add 3 bytes to total length cat21
		BCat21[PPloadC21] = (byte)((TOD >> 16) & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)((TOD >> 8) & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(TOD & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer
				
	} else if(mm.msgtype == 5 || mm.msgtype == 21) {
		// fsd1
		BCat21[c21fsd1] |= 0x80 // Data Source Identifier SAC/SIC
		BCat21[c21len] += 2     // add 2 bytes to total length cat21
		BCat21[PPloadC21] = (byte)(SAC)   // System Area Code
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(SIC)   // System Identification Code
		PPloadC21 += 1          // add pointer

		BCat21[c21fsd1] |= 0x40 // Target Report Descriptor
		BCat21[c21len] += 2     // add 2 bytes to total length cat21
		BCat21[PPloadC21] = 0x01   // 24 bit ICAO Address
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = 0x00   // Actual Target Report
		PPloadC21 += 1          // add pointer

		//fsd2
		BCat21[c21fsd2] |= 0x10 // Target Address
		BCat21[c21len] += 3     // add 3 bytes to total length cat21
		BCat21[PPloadC21] = (byte)(mm.icao1)   // ICAO
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(mm.icao2)   // ICAO
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(mm.icao3)   // ICAO
		PPloadC21 += 1          // add pointer

		//fsd3
		BCat21[c21fsd3] |= 0x20 // Quality Indicator
		BCat21[c21len] += 1     // add 1 byte to total length cat21
		BCat21[PPloadC21] = 0   // Quality
		PPloadC21 += 1          // add pointer

		BCat21[c21fsd3] |= 0x08 // Mode 3/A Code
		BCat21[c21len] += 2     // add 2 byte to total length cat21
		BCat21[PPloadC21] = (byte)((mm.squawk >> 8) & 0xff)   // Mode 3/A (Squawk Number)
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(mm.squawk & 0xff)
		PPloadC21 += 1          // add pointer
		
		//fsd4
		TOD := (int64)(mm.daysec) * 128 + (int64)(mm.nanosec) / 7812500
		BCat21[c21fsd4] |= 0x02 // Time of report transmission
		BCat21[c21len] += 3     // add 3 bytes to total length cat21
		BCat21[PPloadC21] = (byte)((TOD >> 16) & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)((TOD >> 8) & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer
		BCat21[PPloadC21] = (byte)(TOD & 0xFF)   // TOD
		PPloadC21 += 1          // add pointer

	} else if(mm.msgtype == 17) {
		/* DF 17 */
		if(mm.metype >= 1 && mm.metype <= 4) {
			/* Aircraft Identification */

			// fsd1
			BCat21[c21fsd1] |= 0x80 // Data Source Identifier SAC/SIC
			BCat21[c21len] += 2     // add 2 bytes to total length cat21
			BCat21[PPloadC21] = (byte)(SAC)   // System Area Code
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(SIC)   // System Identification Code
			PPloadC21 += 1          // add pointer

			BCat21[c21fsd1] |= 0x40 // Target Report Descriptor
			BCat21[c21len] += 2     // add 2 bytes to total length cat21
			BCat21[PPloadC21] = 0x01   // 24 bit ICAO Address
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = 0x00   // Actual Target Report
			PPloadC21 += 1          // add pointer

			//fsd2
			BCat21[c21fsd2] |= 0x10 // Target Address
			BCat21[c21len] += 3     // add 3 bytes to total length cat21
			BCat21[PPloadC21] = (byte)(mm.icao1)   // ICAO
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(mm.icao2)   // ICAO
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(mm.icao3)   // ICAO
			PPloadC21 += 1          // add pointer

			//fsd3
			BCat21[c21fsd3] |= 0x20 // Quality Indicator
			BCat21[c21len] += 1     // add 1 byte to total length cat21
			BCat21[PPloadC21] = 0   // Quality
			PPloadC21 += 1          // add pointer

			//fsd4
			TOD := (int64)(mm.daysec) * 128 + (int64)(mm.nanosec) / 7812500
			BCat21[c21fsd4] |= 0x02 // Time of report transmission
			BCat21[c21len] += 3     // add 3 bytes to total length cat21
			BCat21[PPloadC21] = (byte)((TOD >> 16) & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((TOD >> 8) & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(TOD & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer

			//fsd5
			var target_id uint64 = 0
			for i := 0; i < 8; i++ {
				ti := mm.identification[i]
				target_id = target_id << 6 + (uint64)(ti)
			}
			BCat21[c21fsd5] |= 0x80 // Target Identification
			BCat21[c21len] += 6     // add 6 byte to total length cat21
			BCat21[PPloadC21] = (byte)((target_id >> 40) & 0xFF)  // Target Identification
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((target_id >> 32) & 0xFF)  // Target Identification
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((target_id >> 24) & 0xFF)  // Target Identification
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((target_id >> 16) & 0xFF)  // Target Identification
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((target_id >> 8) & 0xFF)  // Target Identification
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(target_id & 0xFF)  // Target Identification
			PPloadC21 += 1          // add pointer

			//TODO: Aircraft Type
			//fmt.Printf("    Aircraft Type   : %s\n", ac_type_str[mm.aircraft_type])
		} else if(mm.metype >= 9 && mm.metype <= 18) {
			// fsd1
			BCat21[c21fsd1] |= 0x80 // Data Source Identifier SAC/SIC
			BCat21[c21len] += 2     // add 2 bytes to total length cat21
			BCat21[PPloadC21] = (byte)(SAC)   // System Area Code
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(SIC)   // System Identification Code
			PPloadC21 += 1          // add pointer

			BCat21[c21fsd1] |= 0x40 // Target Report Descriptor
			BCat21[c21len] += 2     // add 2 bytes to total length cat21
			BCat21[PPloadC21] = 0x01   // 24 bit ICAO Address
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = 0x00   // Actual Target Report
			PPloadC21 += 1          // add pointer

			if(mm.cpr_decoded == 1) {
				lat := (int64)(mm.latitude * math.Pow(2, 30) / 180)
				lon := (int64)(mm.longitude * math.Pow(2, 30) / 180)
				BCat21[c21fsd1] |= 0x02 // Position in WGS-84 coordinates high res
				BCat21[c21len] += 8     // add 8 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((lat >> 24) & 0xFF)   // Latitude
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((lat >> 16) & 0xFF)
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((lat >> 8) & 0xFF)
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(lat & 0xFF)
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((lon >> 24) & 0xFF)   // Longitude
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((lon >> 16) & 0xFF)
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((lon >> 8) & 0xFF)
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(lon & 0xFF)
				PPloadC21 += 1          // add pointer
			}

			//fsd2
			BCat21[c21fsd2] |= 0x10 // Target Address
			BCat21[c21len] += 3     // add 3 bytes to total length cat21
			BCat21[PPloadC21] = (byte)(mm.icao1)   // ICAO
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(mm.icao2)   // ICAO
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(mm.icao3)   // ICAO
			PPloadC21 += 1          // add pointer

			time_hp := (uint64)(mm.sec) * (uint64)(math.Pow(2, 30)) + (uint64)(mm.nanosec) * (uint64)(math.Pow(2, 30)) / 1000000000 
			BCat21[c21fsd2] |= 0x04 // Time of message reception of position - high precision
			BCat21[c21len] += 4     // add 3 bytes to total length cat21
			BCat21[PPloadC21] = (byte)((time_hp >> 24) & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((time_hp >> 16) & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((time_hp >> 8) & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(time_hp & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer

			//fsd3
			BCat21[c21fsd3] |= 0x20 // Quality Indicator
			BCat21[c21len] += 4     // add 1 byte to total length cat21
			BCat21[PPloadC21] = (byte)(mm.NIC) << 1 | 0x01   // Quality
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = 1   // 1st ext
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = 1   // 2nd ext
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(mm.PIC) << 4   // 3rd ext
			PPloadC21 += 1          // add pointer

			FL := mm.altitude * 4 / 100		
			BCat21[c21fsd3] |= 0x02 // Flight Level
			BCat21[c21len] += 2     // add 1 byte to total length cat21
			BCat21[PPloadC21] = (byte)((FL >> 8) & 0xff)   // Flight Level
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(FL & 0xff)   // Flight Level
			PPloadC21 += 1          // add pointer

			//fsd4
			TOD := (int64)(mm.daysec) * 128 + (int64)(mm.nanosec) / 7812500
			BCat21[c21fsd4] |= 0x02 // Time of report transmission
			BCat21[c21len] += 3     // add 3 bytes to total length cat21
			BCat21[PPloadC21] = (byte)((TOD >> 16) & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)((TOD >> 8) & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer
			BCat21[PPloadC21] = (byte)(TOD & 0xFF)   // TOD
			PPloadC21 += 1          // add pointer

		} else if(mm.metype >= 19 && mm.mesub >= 1 && mm.mesub <= 4) {
			if(mm.mesub == 1 || mm.mesub == 2) {
				// fsd1
				BCat21[c21fsd1] |= 0x80 // Data Source Identifier SAC/SIC
				BCat21[c21len] += 2     // add 2 bytes to total length cat21
				BCat21[PPloadC21] = (byte)(SAC)   // System Area Code
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(SIC)   // System Identification Code
				PPloadC21 += 1          // add pointer

				BCat21[c21fsd1] |= 0x40 // Target Report Descriptor
				BCat21[c21len] += 2     // add 2 bytes to total length cat21
				BCat21[PPloadC21] = 0x01   // 24 bit ICAO Address
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = 0x00   // Actual Target Report
				PPloadC21 += 1          // add pointer

				//fsd2
				BCat21[c21fsd2] |= 0x10 // Target Address
				BCat21[c21len] += 3     // add 3 bytes to total length cat21
				BCat21[PPloadC21] = (byte)(mm.icao1)   // ICAO
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(mm.icao2)   // ICAO
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(mm.icao3)   // ICAO
				PPloadC21 += 1          // add pointer

				//fsd3
				time_hp := (uint64)(mm.sec) * (uint64)(math.Pow(2, 30)) + (uint64)(mm.nanosec) * (uint64)(math.Pow(2, 30)) / 1000000000 
				BCat21[c21fsd3] |= 0x80 // Time of message reception of velocity - high precision
				BCat21[c21len] += 4     // add 3 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((time_hp >> 24) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((time_hp >> 16) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((time_hp >> 8) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(time_hp & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer

				BCat21[c21fsd3] |= 0x20 // Quality Indicator
				BCat21[c21len] += 1     // add 1 byte to total length cat21
				BCat21[PPloadC21] = (byte)(mm.NAC) << 5    // Quality
				PPloadC21 += 1          // add pointer

				//fsd4
				vert_rate := (int32) (mm.vert_rate * 4 / 25)
				if(mm.vert_rate_sign == 1) {
					vert_rate = vert_rate ^ 0x7fff + 1
				}
				BCat21[c21fsd4] |= 0x20 // Barometric vertical rate
				BCat21[c21len] += 2     // add 2 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((vert_rate >> 8) & 0xFF) //
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(vert_rate & 0xFF) //
				PPloadC21 += 1          // add pointer

				groundspeed := (int64) (mm.velocity * math.Pow(2,14) / 3600)
				heading := (int64)(mm.heading * math.Pow(2,16) / 360)
				BCat21[c21fsd4] |= 0x08 // Airborne Ground Vector
				BCat21[c21len] += 4     // add 4 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((groundspeed >> 8) & 0xFF) // ground speed
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(groundspeed & 0xFF)
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((heading >> 8) & 0xFF) // track angle
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(heading & 0xFF)
				PPloadC21 += 1          // add pointer

				TOD := (int64)(mm.daysec) * 128 + (int64)(mm.nanosec) / 7812500
				BCat21[c21fsd4] |= 0x02 // Time of report transmission
				BCat21[c21len] += 3     // add 3 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((TOD >> 16) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((TOD >> 8) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(TOD & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer

			} else if(mm.mesub == 3 || mm.mesub == 4) {
				// fsd1
				BCat21[c21fsd1] |= 0x80 // Data Source Identifier SAC/SIC
				BCat21[c21len] += 2     // add 2 bytes to total length cat21
				BCat21[PPloadC21] = (byte)(SAC)   // System Area Code
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(SIC)   // System Identification Code
				PPloadC21 += 1          // add pointer

				BCat21[c21fsd1] |= 0x40 // Target Report Descriptor
				BCat21[c21len] += 2     // add 2 bytes to total length cat21
				BCat21[PPloadC21] = 0x01   // 24 bit ICAO Address
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = 0x00   // Actual Target Report
				PPloadC21 += 1          // add pointer

				//fsd2
				if(mm.airspeed_type == 0) {
					//Indicated Airspeed
					airspeed := (int64)((float64)(mm.airspeed) * math.Pow(2,14) / 3600)
					BCat21[c21fsd4] |= 0x40 // Air Speed
					BCat21[c21len] += 2     // add 2 bytes to total length cat21
					BCat21[PPloadC21] = (byte)((airspeed >> 8) & 0xFF) // ground speed
					PPloadC21 += 1          // add pointer
					BCat21[PPloadC21] = (byte)(airspeed & 0xFF)
					PPloadC21 += 1          // add pointer
				} else {
					//True Airspeed
					airspeed := (int64)(mm.airspeed)
					BCat21[c21fsd4] |= 0x20 // True Air Speed
					BCat21[c21len] += 2     // add 2 bytes to total length cat21
					BCat21[PPloadC21] = (byte)((airspeed >> 8) & 0xFF) // ground speed
					PPloadC21 += 1          // add pointer
					BCat21[PPloadC21] = (byte)(airspeed & 0xFF)
					PPloadC21 += 1          // add pointer
				}

				BCat21[c21fsd2] |= 0x10 // Target Address
				BCat21[c21len] += 3     // add 3 bytes to total length cat21
				BCat21[PPloadC21] = (byte)(mm.icao1)   // ICAO
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(mm.icao2)   // ICAO
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(mm.icao3)   // ICAO
				PPloadC21 += 1          // add pointer

				//fsd3
				time_hp := (uint64)(mm.sec) * (uint64)(math.Pow(2, 30)) + (uint64)(mm.nanosec) * (uint64)(math.Pow(2, 30)) / 1000000000 
				BCat21[c21fsd3] |= 0x80 // Time of message reception of velocity - high precision
				BCat21[c21len] += 4     // add 3 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((time_hp >> 24) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((time_hp >> 16) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((time_hp >> 8) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(time_hp & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer

				BCat21[c21fsd3] |= 0x20 // Quality Indicator
				BCat21[c21len] += 1     // add 1 byte to total length cat21
				BCat21[PPloadC21] = (byte)(mm.NAC) << 5    // Quality
				PPloadC21 += 1          // add pointer

				//fsd4
				heading := (int64)(mm.heading * math.Pow(2,16) / 360)
				BCat21[c21fsd4] |= 0x80 // Magnetic Heading
				BCat21[c21len] += 2     // add 2 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((heading >> 8) & 0xFF) // track angle
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(heading & 0xFF)
				PPloadC21 += 1          // add pointer

				vert_rate := (int32) (mm.vert_rate * 4 / 25)
				if(mm.vert_rate_sign == 1) {
					vert_rate = vert_rate ^ 0x7fff + 1
				}
				BCat21[c21fsd4] |= 0x20 // Barometric vertical rate
				BCat21[c21len] += 2     // add 2 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((vert_rate >> 8) & 0xFF) //
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(vert_rate & 0xFF) //
				PPloadC21 += 1          // add pointer


				TOD := (int64)(mm.daysec) * 128 + (int64)(mm.nanosec) / 7812500
				BCat21[c21fsd4] |= 0x02 // Time of report transmission
				BCat21[c21len] += 3     // add 3 bytes to total length cat21
				BCat21[PPloadC21] = (byte)((TOD >> 16) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)((TOD >> 8) & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer
				BCat21[PPloadC21] = (byte)(TOD & 0xFF)   // TOD
				PPloadC21 += 1          // add pointer

			}
		}
	}

	BCat21var := make([]byte, BCat21[c21len])
	copy(BCat21var,BCat21)
	return BCat21var
}


func main() {
	l := new(List)
	var found_node *Node

	config_file := "adsb.conf"
	if len(os.Args) > 1 {
		config_file = os.Args[1]
	}
	file,err := ini.LoadFile(config_file)
	if err != nil { fmt.Printf("Error: %s", err) }
	adsb_server, ok := file.Get("default", "adsb_server")
	if !ok {panic("adsb_server configuration missing")}
	udp_server, ok := file.Get("default", "udp_server")
	if !ok {panic("udp_server configuration missing")}

	sac, ok := file.Get("default", "SAC")
	if !ok {panic("SAC configuration missing")}
	sic, ok := file.Get("default", "SIC")
	if !ok {panic("SIC configuration missing")}
	SAC, err := strconv.ParseInt(sac, 10, 32)
	if err != nil {
		fmt.Print("error")
	}

	SIC, err := strconv.ParseInt(sic, 10, 32)
	if err != nil {
		fmt.Print("error")
	}


	udpserver := strings.Split(udp_server, ",")
	for i := range udpserver {
		udp[i], err = net.Dial("udp", udpserver[i])
		if err != nil {
			fmt.Printf("Some error %v", err)
			return
		}
		fmt.Println("Connected to UDP server " + udpserver[i])
	}

	/* connect to this socket */
	conn, _ := net.Dial("tcp", adsb_server + ":10003")
	for {
		/* listen for reply */
		message := make([]byte, 100)
		n, _ := conn.Read(message)

		/* remove trailing escape char */
		for i:= 2; i < n - 1; i++ {
			if message[i] == 0x1A {
				for j := i + 1; j < n - 1; j++ {
					message[j] = message[j + 1]
				}
				n--
			}
		}

		if(message[1] == 0x32 || message[1] == 0x33) {
			mm.daysec = ((uint64)(message[2]) << 10) | ((uint64)(message[3]) << 2) | ((uint64)(message[4]) >> 6)
			mm.nanosec = (((uint64)(message[4]) & (0x3f)) << 24) | ((uint64)(message[5]) << 16) | ((uint64)(message[6]) << 8) | (uint64)(message[7])

			mm.time = mm.daysec * 1000 + mm.nanosec / 1000000
			mm.hour = mm.time / 3600000
			temp := mm.time - mm.hour * 3600000
			mm.min = temp / 60000
			temp = temp - mm.min * 60000
			mm.sec = temp / 1000
			temp = temp - mm.sec * 1000
			mm.sec1000 = temp

			/* extract Mode S frame from message */
			for i := 9; i < n; i++ {
				message[i - 9] = message[i]
			}
			n = n - 9

			mm.msg = message
			mm.length = n

			mm.msgtype = (int)(message[0]) >> 3
			mm.msgbits = modesMessageLenByType(mm.msgtype)

			mm.crc = ((uint32)(message[mm.msgbits / 8 - 3]) << 16) |
					 ((uint32)(message[mm.msgbits / 8 - 2]) << 8) |
					 (uint32)(message[mm.msgbits / 8 - 1])

			mm.ca = (int)(message[0]) & 7

			mm.icao1 = (int)(message[1])
			mm.icao2 = (int)(message[2])
			mm.icao3 = (int)(message[3])

			// DF17 type (Assuming)
			mm.metype = (int)(message[4]) >> 3		/* Extended squitter message type */
			mm.mesub = (int)(message[4]) & 7		/* Extended squitter message subtype */

			// Field for DF4, 5, 20, 21
			mm.fs = (int)(message[0]) & 7										/* Flight status */
			mm.dr = ((int)(message[1]) >> 3) & 31								/* Request extraction of downlink request */
			mm.um = (((int)(message[1]) & 7) << 3) | ((int)(message[2]) >> 5)	/* Request extraction of downlink request */

			a := (int)(((message[3] & 0x80) >> 5) |
				 ((message[2] & 0x02) >> 0) |
				 ((message[2] & 0x08) >> 3))
			b := (int)(((message[3] & 0x02) << 1) |
				 ((message[3] & 0x08) >> 2) |
				 ((message[3] & 0x20) >> 5))
			c := (int)(((message[2] & 0x01) << 2) |
				 ((message[2] & 0x04) >> 1) |
				 ((message[2] & 0x10) >> 4))
			d := (int)(((message[3] & 0x01) << 2) |
				 ((message[3] & 0x04) >> 1) |
				 ((message[3] & 0x10) >> 4))
			mm.identity = a * 1000 + b * 100 + c * 10 + d
			mm.squawk = a << 9 + b << 6 + c << 3 + d

			/* Fix ICAO address for DFs with an AP field (xored addr and crc). */
			if (mm.msgtype != 11 && mm.msgtype != 17) {
				msgtype := mm.msgtype
				msgbits := mm.msgbits
				if (msgtype == 0 ||
				 msgtype == 4 ||
				 msgtype == 5 ||
				 msgtype == 16 ||
				 msgtype == 20 ||
				 msgtype == 21 ||
				 msgtype == 24) {
					lastbyte := msgbits/8 - 1
					aux := message
					crc := modesChecksum(aux, msgbits)
					aux[lastbyte] ^= (byte) (crc & 0xff)
					aux[lastbyte-1] ^= (byte) ((crc >> 8) & 0xff)
					aux[lastbyte-2] ^= (byte) ((crc >> 16) & 0xff)
					mm.icao1 = (int)(aux[lastbyte - 2])
					mm.icao2 = (int)(aux[lastbyte - 1])
					mm.icao3 = (int)(aux[lastbyte])
				}
			}

			/* Decode 13 bil altitude for DF0, DF4, DF16, DF20 */
			if (mm.msgtype == 0 || mm.msgtype == 4 || mm.msgtype == 16 || mm.msgtype == 20) {
				m_bit := (int)(message[3] & (1 << 6))
				q_bit := (int)(message[3] & (1 << 4))
				if (m_bit == 0) {
					mm.unit = 0			/* modes unit feet */
					n := (((int)(message[2]) & 31) << 6) |
						 (((int)(message[3]) & 0x80) >> 2) |
						 (((int)(message[3]) & 0x20) >> 1) |
						 ((int)(message[3]) & 15)
					if (q_bit != 0) {
						mm.altitude = n * 25 - 1000
					} else {
						mm.altitude = n * 100 - 1000
					}
				} else {
					mm.unit = 1			/* modes unit meter */
					mm.altitude = 0
				}
			}  

			/* Decode extended squitter specific stuff */
			if (mm.msgtype == 17) {
				/* Decode the extended squitter message */
				if (mm.metype >= 1 && mm.metype <= 4) {
					/* Aircraft Identification and Category */
					mm.aircraft_type = mm.metype - 1
					mm.identification[0] = (byte) (message[5] >> 2)
					mm.identification[1] = (byte) (((message[5] & 3) << 4) | (message[6] >> 4))
					mm.identification[2] = (byte) (((message[6] & 15) << 2) | (message[7] >> 6))
					mm.identification[3] = (byte) (message[7] & 63)
					mm.identification[4] = (byte) (message[8] >> 2)
					mm.identification[5] = (byte) (((message[8] & 3) << 4) | (message[9] >> 4))
					mm.identification[6] = (byte) (((message[9] & 15) << 2) | (message[10] >> 6))
					mm.identification[7] = (byte) (message[10] & 63)

					mm.flight[0] = ais_charset[(int) (message[5] >> 2)]
					mm.flight[1] = ais_charset[(int) (((message[5] & 3) << 4) | (message[6] >> 4))]
					mm.flight[2] = ais_charset[(int) (((message[6] & 15) << 2) | (message[7] >> 6))]
					mm.flight[3] = ais_charset[(int) (message[7] & 63)]

					mm.flight[4] = ais_charset[(int) (message[8] >> 2)]
					mm.flight[5] = ais_charset[(int) (((message[8] & 3) << 4) | (message[9] >> 4))]
					mm.flight[6] = ais_charset[(int) (((message[9] & 15) << 2) | (message[10] >> 6))]
					mm.flight[7] = ais_charset[(int) (message[10] & 63)]
					mm.flight[8] = 0x00
				} else if (mm.metype >= 9 && mm.metype <= 18) {
					/* Airborne position Message */
					mm.sbnic = (int) (message[4] & 1)
					mm.fflag = (int) ((message[6] & 0x04) >> 2)
					mm.tflag = (int) ((message[6] & 0x08) >> 3)

					q_bit := (int) (message[5] & 1)
					mm.unit = 0			/* modes unit feet */
					n := (((int)(message[5]) >> 1) << 4) |
						 (((int)(message[6]) & 0xF0) >> 4)
					if (q_bit != 0) {
						mm.altitude = n * 25 - 1000
					} else {
						mm.altitude = n * 100 - 1000
					}

					mm.raw_latitude = (((int)(message[6]) & 3) << 15) |
									   ((int)(message[7]) << 7) |
									   ((int)(message[8]) >> 1)
					mm.raw_longitude = (((int)(message[8]) & 1) << 16) |
										((int)(message[9]) << 8) |
										((int)(message[10]))
					mm.latitude = 0
					mm.longitude = 0
					mm.NIC, mm.PIC = decodeNIC(mm.metype, mm.sbnic)

				} else if (mm.metype == 19 && mm.mesub >=1 && mm.mesub<= 4) {
					mm.NAC = (int)((message[5] & 0x38) >> 3)
					/* Airborne velocity Message */
					if (mm.mesub == 1 || mm.mesub == 2) {
						mm.ew_dir = (int)((message[5] & 4) >> 2)
						mm.ew_velocity = (((int)(message[5]) & 3) << 8) | (int)(message[6])  
						mm.ns_dir = (int)((message[7] & 0x80) >> 7)
						mm.ns_velocity = (((int)(message[7]) & 0x7f) << 3) | (((int)(message[8]) & 0xe0) >> 5)
						mm.vert_rate_source = (int)((message[8] & 0x10) >> 4)
						mm.vert_rate_sign = (int)((message[8] & 0x08) >> 3)
						mm.vert_rate = ((((int)(message[8]) & 7) << 6) | (((int)(message[9]) & 0xfc) >> 2) - 1) * 64
						/* Compute velocity and angle from the two speed component */
						mm.velocity = math.Sqrt(math.Pow((float64)(mm.ns_velocity), 2) + math.Pow((float64)(mm.ew_velocity), 2))
						if (mm.velocity != 0) {
							ewv := mm.ew_velocity
							nsv := mm.ns_velocity
							if (mm.ew_dir == 1) { ewv *= -1 }
							if (mm.ns_dir == 1) { nsv *= -1 }
							heading := math.Atan2((float64)(ewv), (float64)(nsv))
							mm.heading = heading * 360 / (math.Pi * 2)
							if (mm.heading < 0) { mm.heading += 360 }
						} else {
							mm.heading = 0
						}
					} else if (mm.mesub == 3 || mm.mesub == 4) {
						mm.heading_is_valid = (int)(message[5] & (1 << 2))
						mm.heading = (360 / 128) * (float64)((message[5] & 3 << 5) | (message[6] >> 3))
						mm.airspeed_type = ((int)(message[7]) >> 7)
						mm.airspeed = (((int)(message[7]) & 0x7f) << 3) | (((int)(message[8]) & 0xe0) >> 5)
						mm.vert_rate_source = (int)((message[8] & 0x10) >> 4)
						mm.vert_rate_sign = (int)((message[8] & 0x08) >> 3)
						mm.vert_rate = ((((int)(message[8]) & 7) << 6) | (((int)(message[9]) & 0xfc) >> 2) - 1) * 64						
					}
				}
			}


			if (mm.msgtype == 0 || mm.msgtype == 4 || mm.msgtype == 20) {
				addr := ((uint32)(mm.icao1) << 16) | ((uint32)(mm.icao2) << 8) | (uint32)(mm.icao3)

				/* Lookup aircraft or create a new one */
				found_node = l.Find(addr)
				if found_node == nil {
					var a Aircraft

					a.addr = addr
					a.altitude = mm.altitude
					a.seen = time.Now()

					l.Push(a)
				} else {
					a := &(found_node.Aircraft)
					a.altitude = mm.altitude
					a.seen = time.Now()
				}
			}
			/* Decoding DF 17 Airborne position Message CPR Latitude Longitude */
			if (mm.msgtype == 17) {
				if (mm.metype >= 1 && mm.metype <=4) {
					addr := ((uint32)(mm.icao1) << 16) | ((uint32)(mm.icao2) << 8) | (uint32)(mm.icao3)

					/* Lookup aircraft or create a new one */
					found_node = l.Find(addr)
					if found_node == nil {
						var a Aircraft

						a.addr = addr
						a.flight = string(mm.flight[:9])
						a.seen = time.Now()

						l.Push(a)
					} else {
						a := &(found_node.Aircraft)
						a.flight = string(mm.flight[:9])
						a.seen = time.Now()
					}
				} else if (mm.metype >= 9 && mm.metype <= 18) {
					addr := ((uint32)(mm.icao1) << 16) | ((uint32)(mm.icao2) << 8) | (uint32)(mm.icao3)

					/* Lookup aircraft or create a new one */
					found_node = l.Find(addr)
					if found_node == nil {
						var a Aircraft

						a.addr = addr
						if(mm.fflag == 1) { /* Odd CPR */
							a.odd_cprlat = mm.raw_latitude
							a.odd_cprlon = mm.raw_longitude
							a.odd_cprtime = mm.time
						} else {	/* Even CPR */
							a.even_cprlat = mm.raw_latitude
							a.even_cprlon = mm.raw_longitude
							a.even_cprtime = mm.time
						}
						a.seen = time.Now()

						l.Push(a)
					} else {
						a := &(found_node.Aircraft)
						if(mm.fflag == 1) { /* Odd CPR */
							a.odd_cprlat = mm.raw_latitude
							a.odd_cprlon = mm.raw_longitude
							a.odd_cprtime = mm.time
						} else {	/* Even CPR */
							a.even_cprlat = mm.raw_latitude
							a.even_cprlon = mm.raw_longitude
							a.even_cprtime = mm.time
						}
						a.seen = time.Now()

						/* If the two data is less than 10 second apart, compute the position */
						mm.cpr_decoded = 0
						if(math.Abs((float64)(a.even_cprtime - a.odd_cprtime)) <= 10000) {
							decodeCPR(a)
							mm.cpr_decoded = 1
							mm.latitude = a.lat
							mm.longitude = a.lon
						}
					}
				} else if(mm.metype == 19) {
					addr := ((uint32)(mm.icao1) << 16) | ((uint32)(mm.icao2) << 8) | (uint32)(mm.icao3)

					/* Lookup aircraft or create a new one */
					found_node = l.Find(addr)
					if found_node == nil {
						var a Aircraft

						a.addr = addr
						a.vrate = mm.vert_rate
						if(mm.vert_rate_sign == 1) {a.vrate *= -1}
						if (mm.mesub == 1 || mm.mesub == 2) {
							a.speed = (int)(mm.velocity)
						} else {
							a.speed = mm.airspeed
						}
						a.track = (int)(mm.heading)
						a.seen = time.Now()

						l.Push(a)
					} else {
						a := &(found_node.Aircraft)
						a.vrate = mm.vert_rate
						if(mm.vert_rate_sign == 1) {a.vrate *= -1}
						if (mm.mesub == 1 || mm.mesub == 2) {
							a.speed = (int)(mm.velocity)
						} else {
							a.speed = mm.airspeed
						}
						a.track = (int)(mm.heading)
						a.seen = time.Now()
					}
				}
			}

			/* Remove Stale Aircrafts */
			for n := l.First(); n != nil; n = n.Next() {
				if(n != nil) {
					stale_time := time.Since(n.Aircraft.seen)
					address := n.Aircraft.addr
					if (stale_time.Nanoseconds() / 1000000) > AIRCRAFT_TTL  {
						fmt.Printf("Stale Time %d ms\n", stale_time.Nanoseconds()/1000)
						fmt.Printf("Deleting Aircraft %06X\n", address)
						l.Delete(address)
					}
				}
			}

			displayModesMessage(mm)
			if (mm.msgtype == 0) || (mm.msgtype == 4) || (mm.msgtype == 5) || (mm.msgtype == 20) || (mm.msgtype == 21) {
				BCat21 := decodeCAT21(mm, (int32)(SAC), (int32)(SIC))
				for i := range udpserver {
					udp[i].Write(BCat21)
				}
			}


			if (mm.msgtype == 17) {
				if(mm.metype >= 9 && mm.metype <= 18) {
					if(mm.cpr_decoded == 1) {
						BCat21 := decodeCAT21(mm, (int32)(SAC), (int32)(SIC))
						for i := range udpserver {
							udp[i].Write(BCat21)
						}
					}
				} else {
					BCat21 := decodeCAT21(mm, (int32)(SAC), (int32)(SIC))
					for i := range udpserver {
						udp[i].Write(BCat21)
					}
				}
			}

//			displayAircrafts(l)
		}
	}
}
