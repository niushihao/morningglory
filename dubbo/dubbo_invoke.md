invoke CnoocMemberReadFacade.pagingMemberInfo({"class":"cn.com.cnooc.membercenter.member.api.request.MemberPagingRequest","customerName":"123","shopId":null})

invoke CnoocMemberAuditCallbackFacade.callbackForPointRule(1,1)
invoke sayHi({"class":"com.morningglory.dubbo.module.DubboRequest","id":1},null)

invoke cn.com.cnooc.item.item.api.facade.CnoocSaleRuleReadFacade.findEnabledRulesByShopId({"class":"cn.com.cnooc.item.item.api.bean.request.CnoocSaleRuleFindByShopIdRequest","shopId":2100036001})

invoke io.terminus.acl.service.OrganizationService.queryOrgByEmployeeIds([3625,116251],"PRIMARY_DIMENSION",10000000000)
