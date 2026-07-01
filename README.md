<img src="https://raw.githubusercontent.com/iambryant/power-dev-playbook/main/docs/images/power-servers.png" width="50%" alt="Power Servers" />

# Ansible Playbook: IBM Power

[![CI](https://github.com/iambryant/power-dev-playbook/actions/workflows/ci.yml/badge.svg)](https://github.com/iambryant/power-dev-playbook/actions/workflows/ci.yml)

The playbooks in this repository configure my IBM Power infrastructure from the ground up.

## Initial configuration of an IBM Power server

Upon receiving a Power server, I make sure to do these things to get it up and running (Assuming I have an HMC set up with VIOS images uploaded to it):

  - Ensure server is reset/not managed by an HMC
  - Set HMC user password through ASMI
  - Import server from HMC using that HMC user and password
  - Plug in physical disks for the rootvg of the VIOS
  - Run `vios` role
    > [!NOTE]
    > The underlying `ibm.power_hmc.vios` module that the `vios` role uses assumes you are using Fibre Channel-backed storage and doesn't
    > natively support creating a mirrored local rootvg during deployment. Mirroring currently is done manually post-install.
  - Manually create a volume group for logical partitions to use
  - Link aggregate needed interfaces and create virtual networks on top of them
    > [!NOTE]
    > As a bonus, if you include the same physical interface you used for the VIOS install into a link aggregation bundle, VIOS traffic
    > will automatically benefit from the redundancy as well.
  - Create needed logical partitions, then:
    - Assign physical or logical volumes to the logical partition
    - Assign virtual networks to the logical partition
    > [!NOTE]
    > Likewise, this step needs to be done manually. The underlying `ibm.power_hmc.powervm_lpar_instance` module that the `lpar`
    > role uses also assumes you are using Fibre Channel-backed storage. The module offers a `volume_config` parameter where you'd define a
    > physical volume when creating a logical partition. Since I don't own a Fibre Channel array, attempting to pass a physical volume that is a
    > locally attached disk rather than a Fibre Channel LUN prompts this error:
    > ```
    > [ERROR]: Task failed: Module failed: HmcError: b'com.ibm.pmc.templates.common.framework.exception.TemplateException:
    > java.lang.NullPointerException: Cannot invoke "com.ibm.xmlns.powervm.uom.v2012_10.PhysicalVolume$ReservePolicy.getValue()"
    > because the return value of "com.ibm.xmlns.powervm.uom.v2012_10.PhysicalVolume.getReservePolicy()" is null'
    > ```
    > I suspect that this is because the local disk doesn't have the `ReservePolicy` attribute that a Fibre Channel LUN would otherwise have.
  - Run `lpar` role

## Notes

Make sure to set `gather_facts` to `false` when running playbooks that target your HMC. Ansible likes to create temporary directories, and setting
`gather_facts` to `true` enables that and conflicts with the locked-down nature of the HMC.
