provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
}

data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

module "windows_server" {
  source = "./modules/windows_server"

  for_each = var.servers

  resource_group_name = data.azurerm_resource_group.rg.name
  location            = var.location

  vm_name           = each.value.vm_name
  vm_size           = each.value.vm_size
  availability_zone = each.value.availability_zone

  admin_username = var.admin_username
  admin_password = var.admin_password

  subnet_id = var.subnet_id

  os_disk_size_gb = each.value.os_disk_size_gb
  os_disk_type    = each.value.os_disk_type
  os_version      = each.value.os_version

  data_disks = each.value.data_disks

  tags = each.value.tags
}
