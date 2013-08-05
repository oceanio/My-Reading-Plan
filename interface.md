Interface 
=================

### Company Account Sign in

request: 

    POST /api/company_signin
    data : {
        company     : "",           //string, company name
        username    : "",           //string, user name
        password    : "",           //string, password encoded in md5
        is_redirect : true          //bool, whether to redirect if login success
    }

response: 

    if is_redirect:
        HTTP 302                    //redirect, browser will process this ret
    else:
        data : {
            ret_code    : 0,        //int, 0 for success; other for failure
            err_str     : "",       //string, error description
        }

### Company Account Sign out

request: 

    GET /api/company_signout

response: 

    HTTP 302                        //redirect, browser will process this ret


## Company User Operations

### Company User Add

request: 

    POST /api/company_user_add
    data : {
        company     : "",           //string, company name
        username    : "",           //string, user name
        password    : "",           //string, default password encoded in md5
        attributes  : {
            email       : "",
            telephone   : "",
            mobile      : "",
            group_name  : "",
            ...,                    // any attributes we want
            description : "",
        },        
        roles : [                   //array of string, roles assign to this user
            "admin", 
            "user", 
            ...
        ],                          
        permissions : [             //array of strings, user permissions
            ...
        ]
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        user_id     : 0             //int, user id
    }


### Company User Remove

request: 

    POST /api/company_user_rmv
    data : {
        user_id     : 0,            //int, user id, key to delete a user
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : ""            //string, error description
    }

### Company User Set

request: 

    POST /api/company_user_set
    data : {
        user_id     : 0,            //int, user id, key to edit a user's profile
        attributes  : {
            email       : "",
            telephone   : "",
            mobile      : "",
            group_name  : "",
            ...,                    // any attributes we want
            description : "",
        },        
        roles : [                   //array of string, roles assign to this user
            "admin", 
            "user", 
            ...
        ],
        permissions : [             //array of strings, user permissions
            ...
        ]
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
    }


### Company User Get

request: 

    POST /api/company_user_get
    data : {
        user_id     : 0,            //int, user id, key to get a user's profile
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        user_id     : 0,            //int, user id, key to edit a user's profile
        company     : "",           //string, company name
        username    : "",           //string, user name
        attributes  : {
            email       : "",
            telephone   : "",
            mobile      : "",
            group_name  : "",
            ...,                    // any attributes we want
            description : "",
        },        
        roles : [                   //array of string, roles assign to this user
            "admin", 
            "user", 
            ...
        ],
        permissions : [             //array of strings, user permissions
            ...
        ]
    }

### Company User List

request: 

    POST /api/company_user_list
    data : {
        filter : {
            company     : "",       //string, company name
            group_name  : "",
            ...                     //maybe some other filters
        },
        page_no     : 0,            //current page number
        page_step   : 0,            //how many items you want to show per page
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        page_no     : 0,            //current page number
        prev_page   : 0,            //prev page no
        next_page   : 0,            //next page no
        last_page   : 0,            //last page no
        user_profiles: [
            {
                user_id     : 0,            //int, user id, key to edit a user's profile
                company     : "",           //string, company name
                username    : "",           //string, user name
                attributes  : {
                    ...,                    // any attributes we want to query
                }
            },
            ...
        ]
    }

## Company Group Operations

### Company Group Add

request: 

    POST /api/company_group_add
    data : {
        company     : "",           //string, company name
        group_name  : "",           //string, group name
        attributes  : {
            ...,                    // any attributes we want
            description : "",
        }
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        group_id    : 0             //int, group id
    }

### Company Group Remove

request: 

    POST /api/company_group_rmv
    data : {
        group_id     : 0,            //int, group id, key to delete a group
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : ""            //string, error description
    }

### Company Group Set

request: 

    POST /api/company_group_set
    data : {
        group_id    : 0,            //int, group id, key to edit a group's profile
        attributes  : {
            ...,                    // any attributes we want
            description : "",
        }
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
    }

### Company Group Get

request: 

    POST /api/company_group_get
    data : {
        group_id    : 0,            //int, group id, key to get a group's profile
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        group_id    : 0,            //int, group id, key to edit a group's profile
        company     : "",           //string, company name
        attributes  : {
            ...,                    // any attributes we want
            description : "",
        }
    }

### Company Groups List

request: 

    POST /api/company_group_list
    data : {
        filter : {
            company     : "",       //string, company name
            ...                     //maybe some other filters
        },
        page_no     : 0,            //current page number
        page_step   : 0,            //how many items you want to show per page
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        page_no     : 0,            //current page number
        prev_page   : 0,            //prev page no
        next_page   : 0,            //next page no
        last_page   : 0,            //last page no
        group_info  : [
            {
                group_id    : 0,            //int, group id, key to edit a group's profile
                company     : "",           //string, company name
                attributes  : {
                    ...,                    // any attributes we want
                    description : "",
                }
            },
            ...
        ]
    }



