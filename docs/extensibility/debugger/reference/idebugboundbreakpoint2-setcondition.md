---
title: "IDebugBoundBreakpoint2::SetCondition | Microsoft Docs"
ms.custom: ""
ms.date: "11/04/2016"
ms.technology: 
  - "vs-ide-sdk"
ms.topic: "conceptual"
f1_keywords: 
  - "IDebugBoundBreakpoint2::SetCondition"
helpviewer_keywords: 
  - "SetCondition method"
  - "IDebugBoundBreakpoint2::SetCondition method"
ms.assetid: 5d366876-efed-43d0-8ea1-dfdb009cbfac
author: "gregvanl"
ms.author: "gregvanl"
manager: douge
ms.workload: 
  - "vssdk"
---
# IDebugBoundBreakpoint2::SetCondition
Sets or changes the condition associated with this bound breakpoint.  
  
## Syntax  
  
```cpp  
HRESULT SetCondition(   
   BP_CONDITION bpCondition  
);  
```  
  
```csharp  
int SetCondition(   
   enum_BP_CONDITION bpCondition  
);  
```  
  
#### Parameters  
 `bpCondition`  
 [in] A value from the [BP_CONDITION](../../../extensibility/debugger/reference/bp-condition.md) enumeration that describes the condition.  
  
## Return Value  
 If successful, returns `S_OK`; otherwise, returns an error code. Returns `E_BP_DELETED` if the state of the bound breakpoint object is set to `BPS_DELETED` (part of the [BP_STATE](../../../extensibility/debugger/reference/bp-state.md) enumeration).  
  
## Remarks  
 Any condition that was previously associated with this breakpoint is lost.  
  
## See Also  
 [IDebugBoundBreakpoint2](../../../extensibility/debugger/reference/idebugboundbreakpoint2.md)   
 [BP_CONDITION](../../../extensibility/debugger/reference/bp-condition.md)   
 [BP_STATE](../../../extensibility/debugger/reference/bp-state.md)