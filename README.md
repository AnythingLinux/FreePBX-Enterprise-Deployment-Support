# FreePBX 17 on Debian 13 Trixie with Tailscale

## Client Deployment, VoIP Testing, SIP Trunking, Call Termination and Call-Center Readiness

> **Deployment status:** FreePBX 17 is documented for Debian 12
> Bookworm. Debian 13 Trixie is not the documented production baseline
> for FreePBX 17. This README therefore treats FreePBX 17 on Debian 13
> as an **experimental / compatibility deployment**. For a supported
> production installation, use a FreePBX-supported operating system and
> follow the current official installation guide. Do not promise
> production stability on Debian 13 until the exact package set, upgrade
> path, modules and call flows have been validated.

------------------------------------------------------------------------

## 1. Project objective

This project establishes a private and testable VoIP platform using:

-   **Debian 13 Trixie** as the requested server operating system.
-   **FreePBX 17** as the PBX administration interface.
-   **Asterisk** as the SIP signalling and media engine.
-   **Tailscale** for private management and secure connectivity between
    approved devices.
-   SIP softphones or IP phones for extension-to-extension testing.
-   An authorized SIP trunk for inbound and outbound public telephone
    calls.
-   Optional call-center, CRM, reporting, recording and monitoring
    applications.

The deployment is designed to progress in stages: operating-system
preparation, private network, PBX installation, extension testing, SIP
trunk integration, call-center configuration, security hardening and
acceptance testing.

### Important terminology

In this document, **call termination** means routing an authorized
outbound call through a licensed/authorized carrier or SIP trunk to a
destination telephone network. It does not mean bypassing carrier
controls, caller-ID rules, destination restrictions or applicable
telecom regulations.

------------------------------------------------------------------------

## 2. Scope of work

### Core PBX

-   Install and configure the operating system and required services.
-   Install FreePBX 17 and Asterisk using a documented, reproducible
    method.
-   Configure the FreePBX administrator interface.
-   Create SIP extensions and endpoint credentials.
-   Configure inbound and outbound routes when a SIP trunk is supplied.
-   Configure voicemail, ring groups, IVR, time conditions and basic
    call recording.
-   Configure CDR (Call Detail Records) and basic operational
    monitoring.
-   Integrate Tailscale for private administration and, where validated,
    private SIP endpoint connectivity.

### VoIP validation

-   Register multiple SIP endpoints.
-   Test internal calls in both directions.
-   Verify two-way audio and DTMF.
-   Test hold, transfer, voicemail, ring groups and IVR.
-   Test inbound and outbound calls using an authorized SIP trunk.
-   Verify caller ID presentation, route selection, call release and CDR
    records.
-   Test operation across different networks and after a
    reboot/reconnection.

### Optional business applications

-   Call-center queues and agent extensions.
-   Agent login/logout and pause states, where supported by the selected
    module/application.
-   Inbound queue routing, overflow and after-hours handling.
-   Outbound campaign/dialer integration, subject to lawful consent and
    local rules.
-   CRM/helpdesk integration and screen-pop workflows.
-   Call recording, quality review and retention controls.
-   Wallboards, supervisor tools and advanced reporting.
-   WebRTC/browser calling, if required and separately validated.
-   Backup, disaster recovery and alerting.

------------------------------------------------------------------------

## 3. Compatibility and production warning

### Debian 13 and FreePBX 17

The official FreePBX 17 installation documentation identifies Debian 12
Bookworm as the supported installation target. Debian 13 Trixie may
require package substitutions or workarounds and may not be covered by
the official installation or upgrade path.

Before proceeding on Debian 13:

1.  Confirm the current FreePBX 17 installation documentation and
    supported OS matrix.
2.  Confirm that the required Asterisk version and PHP/database
    dependencies are supported together.
3.  Confirm that all required FreePBX modules install and update
    correctly.
4.  Test `fwconsole`, Apache, PHP, MariaDB and Asterisk after
    installation and after reboot.
5.  Document any package pinning, patches or manual changes.
6.  Test backup and restore on a separate machine.
7.  Obtain client approval for the experimental OS combination.

**Production recommendation:** If the client requires vendor-supported
operation, use the currently documented FreePBX-supported OS. If Debian
13 is mandatory, treat it as a proof of concept until all acceptance
tests pass and the client accepts the support limitations.

Never run an unreviewed third-party install script as root on a
production server. Review its source, pin versions, verify
checksums/signatures where available and retain a rollback snapshot.

------------------------------------------------------------------------

## 4. Infrastructure and server requirements

The final specification depends on simultaneous calls, codec,
transcoding, recording, conferencing, dialer use, retention and expected
growth. The figures below are starting points, not a capacity guarantee.

  ------------------------------------------------------------------------
  Deployment size           Suggested starting point Typical use
  --------------------- ---------------------------- ---------------------
  Lab / proof of         2 vCPU, 4 GB RAM, 40 GB SSD A few test
  concept                                            extensions; no
                                                     production load

  Small office          4 vCPU, 8 GB RAM, 80--100 GB Basic PBX and a small
                                                 SSD number of concurrent
                                                     calls

  Growing office /       8 vCPU, 16 GB RAM, 150--250 Queues, recording and
  small call center                           GB SSD more concurrent calls

  Larger call center               Capacity-designed Requires measured
                               dedicated/VPS cluster load testing and
                                                     separate
                                                     recording/storage
                                                     planning
  ------------------------------------------------------------------------

For a target such as **250 concurrent calls**, do not select a server
from a generic table alone. Conduct a workload assessment and load test
using the actual codecs, call recording, transcoding, conferences,
dialer behaviour and carrier route. Consider separating the PBX,
database/reporting, recording storage and dialer components. Obtain the
carrier's capacity and CPS (calls per second) limits in writing.

### Server prerequisites

-   Dedicated VPS or physical server with stable CPU performance.
-   Static public IPv4 address for SIP trunk connectivity, if required
    by the carrier.
-   Reliable network path and sufficient bandwidth in both directions.
-   SSD/NVMe storage with an I/O and retention plan.
-   DNS hostname, for example `pbx.example.com`, if public services
    require it.
-   A supported Linux installation method and a recoverable server
    snapshot.
-   NTP/time synchronization.
-   Out-of-band console or provider recovery access.
-   A separate backup destination.

### Client-supplied information

-   Server provider, region, public IP and root/sudo access.
-   Confirmed OS image and exact release.
-   Domain/subdomain and DNS control, if needed.
-   Tailscale tailnet administrator or invitation process.
-   Number of users, extensions, agents and supervisors.
-   Expected busy-hour concurrent calls and calls per second.
-   Inbound/outbound call requirements and destination countries.
-   SIP trunk provider and authorized account details.
-   DIDs, approved caller IDs and provider IP/authentication details.
-   Required codecs, transport and RTP port range from the provider.
-   Recording, retention, privacy and access requirements.
-   Business hours, holidays, IVR prompts and call-flow requirements.
-   CRM/helpdesk/API requirements.
-   Emergency calling requirements and the registered service address,
    where applicable.

Do not send SIP passwords, private keys or administrator passwords in a
public README, ticket or source repository. Use a secure password
manager or an approved secret-sharing method.

------------------------------------------------------------------------

## 5. Network architecture

### Private administration and extension testing

``` text
                 Internet
                    |
             VPS public IPv4
                    |
          +--------------------+
          | Debian 13 server   |
          | FreePBX / Asterisk |
          | Tailscale node     |
          +--------------------+
                    |
              Tailnet VPN
          /         |          \
         /          |           \
 Linux softphone  Android     Remote admin
   extension      softphone    workstation
```

Tailscale provides private connectivity between enrolled devices. It
does **not** replace the SIP trunk, PSTN carrier, PBX configuration,
firewall policy or RTP troubleshooting.

### Public SIP trunk architecture

``` text
SIP softphones / IP phones
          |
      FreePBX / Asterisk
          |
     Authorized SIP trunk
          |
   Carrier / ITSP network
          |
     Public telephone
        networks
```

A deployment can use Tailscale for management while using a
carrier-approved public interface for SIP trunk signalling and media.
The exact design must follow the carrier's IP-authentication, transport,
NAT and media requirements.

### Network ports to plan

Do not open every port to the public Internet. Apply the narrowest
source-IP and interface rules possible.

  -----------------------------------------------------------------------
  Service                 Typical port            Access policy
  ----------------------- ----------------------- -----------------------
  SSH                     TCP 22                  Administrator IPs or
                                                  Tailscale only

  FreePBX HTTPS           TCP 443                 Administrator IPs or
                                                  Tailscale only

  SIP                     UDP/TCP 5060            Only trusted endpoints
                                                  or carrier IPs, as
                                                  required

  SIP TLS                 TCP 5061                Only if configured and
                                                  supported

  RTP media               Configured UDP range    Only the required
                                                  endpoint/carrier paths

  Tailscale               Outbound UDP 41641 and  Follow current
                          UDP 3478/STUN behaviour Tailscale network
                                                  documentation; direct
                                                  connections may use
                                                  other paths

  DNS/NTP                 Standard DNS/NTP        Required trusted
                                                  resolvers/time sources
  -----------------------------------------------------------------------

The RTP range is configurable in Asterisk/FreePBX. Confirm the actual
configured range rather than assuming a default. A successful SIP
registration does not prove that RTP audio will pass.

### Tailscale design notes

-   Enrol only authorized devices in the tailnet.
-   Use device approval, MFA and least-privilege access controls.
-   Use Tailscale ACLs/grants to restrict which users/devices can reach
    PBX management and SIP services.
-   Keep SSH and the FreePBX GUI private wherever possible.
-   Do not expose the Tailscale administration interface or device
    credentials.
-   Verify whether the softphone's SIP and RTP traffic is actually
    routed through Tailscale.
-   Avoid overlapping LAN/VPN subnets.
-   Test MTU, NAT traversal and connectivity on mobile networks.
-   Keep a separate provider-console recovery path if Tailscale is
    unavailable.

------------------------------------------------------------------------

## 6. Software and application inventory

  --------------------------------------------------------------------------
  Component               Role                       Required?
  ----------------------- -------------------------- -----------------------
  Debian 13 Trixie        Server OS requested by     Yes, subject to
                          client                     compatibility warning

  FreePBX 17              PBX web administration     Yes

  Asterisk                SIP/media engine           Yes

  Tailscale               Private network and        Yes for this design
                          administration             

  MariaDB                 FreePBX/Asterisk-related   As required by
                          database service,          installer
                          according to install       
                          method                     

  Apache/PHP              FreePBX web stack,         As required by
                          according to supported     installer
                          install method             

  SIP softphone           Extension testing          Yes

  `sngrep`                SIP message                Recommended
                          troubleshooting            

  Wireshark               Packet and RTP analysis    Recommended

  `tcpdump`               Network capture            Recommended

  Monitoring/alerting     Service and resource       Recommended
                          monitoring                 

  Backup destination      Configuration and data     Yes for production
                          recovery                   
  --------------------------------------------------------------------------

### Optional call-center and business applications

Select applications only after confirming compatibility with the exact
Asterisk/FreePBX version and the required workflow.

  -----------------------------------------------------------------------
  Application / option    Potential use           Notes
  ----------------------- ----------------------- -----------------------
  FreePBX Queues          Basic inbound           Start here for a simple
                          call-center queues      agent workflow

  FreePBX Ring Groups /   Department routing and  Native PBX functions
  IVR                     menus                   

  FreePBX UCP             User portal and         Module/version
                          supported user features dependent

  FOP2                    Operator panel and call Commercial licensing
                          visibility              may apply

  QueueMetrics            Queue reporting and     Check supported
                          wallboards              Asterisk version and
                                                  licensing

  VICIdial                Contact-center /        Separate deployment and
                          predictive-dialer       specialist integration;
                          platform                validate compatibility

  GoAutoDial              Dialer/contact-center   Check current
                          option                  maintenance,
                                                  compatibility and
                                                  security before
                                                  selecting

  CRM/helpdesk            Customer records and    Examples include a
                          agent workflow          compatible CRM or
                                                  ticketing platform;
                                                  integration is
                                                  project-specific

  WebRTC client           Browser-based calling   Requires secure
                                                  WebSocket, TLS, NAT and
                                                  media validation

  Prometheus/Grafana or   Infrastructure          Requires exporters and
  equivalent              dashboards              alert design

  Centralized log         Log retention and       Restrict access to call
  platform                investigation           and customer data
  -----------------------------------------------------------------------

A dialer is not automatically included with FreePBX.
Predictive/progressive dialing, campaign management, consent management
and agent-state workflows may require a separate application and
integration project.

------------------------------------------------------------------------

## 7. Implementation phases

### Phase 1 --- Discovery and design

1.  Confirm Debian 13 is mandatory and obtain written acceptance of the
    compatibility limitation.
2.  Confirm whether the objective is a lab, office PBX or production
    call center.
3.  Record expected extensions, agents, supervisors and peak concurrent
    calls.
4.  Identify inbound/outbound destinations and the SIP carrier.
5.  Collect carrier interconnection specifications.
6.  Agree the network, security, recording and backup design.
7.  Define acceptance tests and change-control/rollback arrangements.

**Deliverable:** Approved implementation plan and requirements
checklist.

### Phase 2 --- Operating-system preparation

-   Install the agreed Debian image.
-   Apply security updates.
-   Set hostname, DNS resolver and timezone.
-   Configure NTP.
-   Create named administrator accounts and SSH key access.
-   Disable password-based root SSH where operationally appropriate.
-   Configure a host firewall with a default-deny inbound policy.
-   Enable a recovery console and take a clean snapshot.
-   Record OS and package versions.

**Deliverable:** Hardened, recoverable base server.

### Phase 3 --- Tailscale deployment

-   Install Tailscale from its official repository/instructions.
-   Authenticate the server into the approved tailnet.
-   Approve the device and apply least-privilege access policy.
-   Install Tailscale on test endpoints.
-   Confirm device-to-device connectivity.
-   Confirm that only intended management/SIP paths are reachable.
-   Test reconnection after reboot and network changes.

**Deliverable:** Approved private network connectivity.

### Phase 4 --- FreePBX 17 and Asterisk

-   Follow the current official FreePBX 17 installation guide.
-   On Debian 13, document every deviation from the supported
    installation procedure.
-   Install the required Asterisk and web/database dependencies using
    compatible versions.
-   Verify the FreePBX console and web interface.
-   Confirm Asterisk and web/database services start automatically.
-   Apply only required FreePBX modules.
-   Set strong administrator credentials and restrict GUI access.
-   Take a snapshot and a verified configuration backup.

**Deliverable:** Accessible PBX with verified core services.

### Phase 5 --- Extensions and internal calling

-   Create at least two test PJSIP extensions.
-   Generate unique, strong SIP secrets.
-   Configure endpoint transport and codecs.
-   Configure softphones using the PBX address reachable through the
    selected network design.
-   Register each extension.
-   Test calls in both directions.
-   Verify two-way audio, DTMF, hold and transfer.
-   Review Asterisk logs and SIP traces for failures.

**Deliverable:** Successful internal extension calling.

### Phase 6 --- SIP trunk and public call termination

Obtain the following from the authorized ITSP/carrier:

-   SIP server/proxy or SBC hostname/IP.
-   Authentication username and password, or IP-authentication
    requirements.
-   Registration details, if registration is used.
-   Permitted signalling transport and port.
-   Allowed source public IP addresses.
-   Supported codecs and DTMF method.
-   RTP/media IP ranges and port requirements.
-   DID numbers and inbound routing details.
-   Approved outbound caller ID(s).
-   Allowed destinations, concurrent-call limit and CPS limit.
-   Billing, rate limits, fraud controls and escalation contacts.

Configuration work:

1.  Create the PJSIP trunk using the provider's documented parameters.
2.  Restrict access to provider IP ranges where IP-based rules are
    applicable.
3.  Configure outbound routes with explicit permitted
    prefixes/destinations.
4.  Configure inbound routes for assigned DIDs.
5.  Set approved caller ID and presentation rules.
6.  Configure codec order and DTMF according to the carrier.
7.  Validate RTP/media connectivity.
8.  Test with carrier-approved test numbers and controlled destinations.
9.  Check CDR, carrier logs and billing records.
10. Obtain provider confirmation that the route is authorized and
    provisioned.

Do not use unverified routes, spoof caller ID, evade destination
restrictions or place unsolicited calls. Enable international or
premium-rate destinations only when the business has a documented need
and the carrier has authorized them.

**Deliverable:** Verified inbound and outbound calls through the
client's authorized carrier.

### Phase 7 --- Call-center configuration

For a basic inbound call center:

-   Create agent extensions and individual accounts.
-   Create queues by department or service.
-   Configure ring strategy and agent timeout.
-   Set queue announcements and hold music with appropriate licensing.
-   Configure business hours, holidays and after-hours destinations.
-   Configure overflow, no-answer and failure destinations.
-   Configure supervisor permissions.
-   Configure call recording only under an approved policy.
-   Configure CDR/queue reports and access restrictions.
-   Test agent login, availability, pause and logout functions supported
    by the selected solution.

For outbound contact-center workflows:

-   Define campaign purpose and approved contact lists.
-   Confirm consent, do-not-call and calling-hour requirements.
-   Set permitted caller IDs and destination prefixes.
-   Apply carrier CPS and concurrency limits.
-   Establish retry limits and contact suppression rules.
-   Test with internal/authorized test destinations before any live
    campaign.
-   Provide opt-out and complaint-handling workflows where applicable.

**Deliverable:** Tested inbound queue and, if in scope, an approved
outbound workflow.

### Phase 8 --- Security, monitoring and backups

-   Restrict PBX GUI and SSH access.
-   Use strong unique passwords and MFA where supported.
-   Keep FreePBX, Asterisk, OS and modules patched using a tested change
    process.
-   Review firewall rules and SIP exposure.
-   Enable appropriate intrusion detection/rate-limiting protections.
-   Monitor CPU, memory, disk, network, Asterisk channels and service
    health.
-   Monitor trunk registration/reachability and failed-call rates.
-   Back up configuration, database, voicemail and recordings according
    to scope.
-   Store backups off-server with encryption and access control.
-   Test restoration, not just backup creation.
-   Define log and recording retention periods.

**Deliverable:** Security and operations checklist with a tested restore
procedure.

------------------------------------------------------------------------

## 8. Testing and acceptance checklist

The project should not be signed off solely because the FreePBX
dashboard loads. Use the following test matrix and record evidence,
timestamps and results.

  -----------------------------------------------------------------------
  ID                      Test                    Expected result
  ----------------------- ----------------------- -----------------------
  NET-01                  Tailscale device        Approved devices can
                          connectivity            reach only permitted
                                                  services

  SYS-01                  Reboot server           Required services
                                                  return automatically

  SYS-02                  Time synchronization    Correct time and
                                                  timezone

  PBX-01                  FreePBX GUI             Authorized
                                                  administrator can sign
                                                  in over approved path

  PBX-02                  Asterisk service        Service is healthy and
                                                  CLI responds

  SIP-01                  Extension registration  Each test endpoint
                                                  registers

  SIP-02                  Internal call A → B     Call connects and
                                                  releases normally

  SIP-03                  Internal call B → A     Reverse direction works

  RTP-01                  Two-way audio           Both endpoints hear
                                                  each other

  RTP-02                  DTMF                    Correct digits are
                                                  received by the far end

  CALL-01                 Hold / resume           Call resumes without
                                                  media failure

  CALL-02                 Blind/attended transfer Transfer works as
                                                  configured

  CALL-03                 Voicemail               No-answer call reaches
                                                  the correct mailbox

  ROUTE-01                Inbound DID             Call reaches the
                                                  assigned destination

  ROUTE-02                Outbound authorized     Carrier completes the
                          call                    approved test call

  ROUTE-03                Caller ID               Only the approved
                                                  caller ID is presented

  ROUTE-04                CDR                     Call record contains
                                                  correct time, duration
                                                  and disposition

  CC-01                   Queue entry             Caller enters the
                                                  correct queue

  CC-02                   Agent answer            Eligible agent receives
                                                  the call

  CC-03                   Queue overflow          Call follows the
                                                  approved fallback

  REC-01                  Recording policy        Recording starts/stops
                                                  and access is
                                                  restricted as approved

  BAK-01                  Backup                  Backup completes and is
                                                  stored off-server

  BAK-02                  Restore test            Configuration can be
                                                  restored on a test
                                                  system

  SEC-01                  External exposure       No unnecessary
                          review                  management/SIP ports
                                                  are publicly exposed
  -----------------------------------------------------------------------

### Performance and capacity testing

For a production target, test gradually in a controlled environment:

-   Establish a baseline with no active calls.
-   Increase concurrent calls in agreed increments.
-   Measure CPU, memory, network throughput, packet loss, jitter and
    latency.
-   Check call setup rate (CPS), answer rate and failed-call rate.
-   Test the actual codecs and whether transcoding is enabled.
-   Test recording and queue/reporting overhead.
-   Confirm carrier concurrency/CPS limits.
-   Stop the test if service quality or system stability degrades.

Use only test endpoints, test numbers and traffic explicitly authorized
by the carrier. Do not generate artificial traffic to public
destinations without written authorization.

------------------------------------------------------------------------

## 9. Bandwidth and media planning

Approximate bandwidth depends on codec, packetization interval,
transport overhead, encryption, tunnelling and network path. As a
planning example, G.711 commonly consumes roughly 80--100 kbit/s per
call **per direction** at the IP layer, but actual figures vary.

For capacity planning:

-   Calculate both inbound and outbound media.
-   Include Tailscale tunnel overhead if media traverses the tailnet.
-   Include signalling, management, recording transfer and other
    application traffic.
-   Leave headroom for bursts and network variation.
-   Measure actual traffic during representative calls.
-   Avoid transcoding unless needed; transcoding adds CPU load.

Do not size a server or bandwidth plan using codec payload bitrate
alone.

------------------------------------------------------------------------

## 10. Operational handover

The client handover should include:

-   Server hostname, OS release and resource specification.
-   Network diagram and approved port/source rules.
-   FreePBX and Asterisk versions.
-   Tailscale device and access-policy handover (no reusable auth keys
    in the README).
-   Extension inventory and secure credential handover.
-   Trunk configuration summary without plaintext secrets.
-   Inbound/outbound route summary.
-   Queue, IVR and business-hours configuration.
-   Backup schedule and restore instructions.
-   Monitoring and escalation contacts.
-   Test results and known limitations.
-   Maintenance/update procedure.
-   List of paid modules, licenses and third-party services.

The client should own the server, domain, tailnet, carrier account,
phone numbers and application subscriptions wherever possible.

------------------------------------------------------------------------

## 11. Exclusions and dependencies

Unless explicitly included in the quotation, the following are separate
scope items:

-   SIP trunk or carrier account fees.
-   DID number rental and call charges.
-   Telecom licensing, regulatory approval and legal advice.
-   Third-party paid modules, dialer, CRM and reporting licenses.
-   Custom CRM/API development.
-   Bulk outbound campaign design or contact-list acquisition.
-   Custom IVR recording, translation and professional voice talent.
-   Number porting and carrier-side provisioning.
-   High-availability clustering and disaster-recovery infrastructure.
-   Independent security audit or penetration test.
-   Guaranteed call capacity before workload testing.
-   24/7 support or ongoing maintenance.

Client dependencies include timely access to the server, DNS, tailnet,
carrier portal, test devices, approved call destinations, business call
flows and authorized contact lists.

------------------------------------------------------------------------

## 12. Useful official documentation

Check current vendor documentation before installation and before
changing firewall or package configuration:

-   FreePBX: https://www.freepbx.org/
-   FreePBX 17 installation:
    https://sangomakb.atlassian.net/wiki/spaces/PG/pages/22183937/Installing+FreePBX+17+on+Debian+12
-   Asterisk documentation: https://docs.asterisk.org/
-   Asterisk PJSIP configuration:
    https://docs.asterisk.org/Configuration/Channel-Drivers/SIP/Configuring-res_pjsip/
-   Tailscale Linux installation:
    https://tailscale.com/kb/1031/install-linux
-   Tailscale access controls: https://tailscale.com/kb/1018/acls/
-   Debian documentation: https://www.debian.org/doc/

If a vendor changes a documentation URL, use that vendor's current
documentation portal rather than relying on an old installation command.

------------------------------------------------------------------------

## 13. Client sign-off

The deployment is ready for acceptance when:

-   [ ] The client has approved the OS compatibility and support
    position.
-   [ ] The PBX and Tailscale services survive a reboot.
-   [ ] Approved extensions register and make two-way internal calls.
-   [ ] Required inbound/outbound trunk tests pass, if a trunk is in
    scope.
-   [ ] Call-center flows pass the agreed test cases, if included.
-   [ ] Security and firewall review is complete.
-   [ ] Backup and restore have been tested.
-   [ ] Documentation and credentials have been handed over securely.
-   [ ] Outstanding limitations and third-party dependencies are
    recorded.

**Project status:** Not production-approved until the client-specific
acceptance tests, carrier verification, security review and restore test
have passed.
