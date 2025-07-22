# ansible-cisco-autoinstall

Use Event-Driven Ansible to plug into the [Cisco Autoinstall](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/syst-mgmt/b-system-management/m_cf-autoinstall-0.html) process for IOS.

This implementation uses [Netbox](https://netboxlabs.com/) to store device data, but that is optional. The TFTP utilities will work without it if you have another way to provide bootstrap configurations.

## Playbooks
**tftp_install_server.yml** - Just installs the TFTP server that ships with RHEL/Fedora.

**tftp_netbox_populate.yml** - Reads Netbox for management interfaces and prepares device-specific bootstrap configurations according to what it finds. Places these configurations on the default TFTP server location.

**tftp_install_watch_service.yml** - Installs a systemd service (on the machine running the TFTP server) for purposes of detecting config downloads from the Cisco autoinstall process. Designed to send a webhook to an [EDA Event Stream](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_decisions/simplified-event-routing) upon detection.

## Rulebooks

**webhook.yml** - Recieves the webhooks from the TFTP watch service; intended to allow Ansible to run a more complete device configuration than what's included in the bootstrap configs.

## Requirements
1. Device information is loaded into Netbox. For the playbooks in this repo, each device should have an interface that's designated "management-only" with an IP address assigned to it.
2. Device information is loaded in DHCP. A static reservation is required for the MAC address of the management port, or if the device doesn't have a management port, the Vlan1 virtual interface. The DHCP server should provide Option 66 with the IP of a server that will run TFTP and Option 12 with the name of the device.
3. Automation Controller has a Job Template available that runs a full configuration of this type of device. It ideally uses Netbox dynamic inventory.

## Setup

1. Establish an EDA Event Stream with Basic Auth applied to it. Get the Event Stream URL and have the Basic Auth credentials available.
2. Create a Rulebook Activation that uses your Event stream with `webhook.yml`. It should be modified to match the correct Job Template.
3. Use `tftp_install_server.yml` to install the TFTP server somewhere that the device will be able to reach.
4. Use `tftp_install_watch_service.yml` - while providing the info from step 1 - to establish the watch service.
5. Use `tftp_netbox_populate.yml` to read the Netbox data and create configs for the TFTP server to serve.

## Process flow
1. Switch turns on with no startup-config, and enters Autoinstall
2. Switch gets DHCP address from reservation, finds out what its name is, and requests a corresponding bootstrap config from TFTP
3. TFTP serves the requested file
4. TFTP Watch Service sees the above in the TFTP log, and sends an event to the EDA Event Stream
5. The EDA Rulebook activation recieves the event and kicks off full provisioning of the new device
