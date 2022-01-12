<template>
    <div class="modal" @click="closeModal()">
        <div class="p-sm roundness-xs bg-white-themed" @click="preventClose = true">
            <div class="group vertical-center">
                <p class="text-medium text-lg">{{ $t("authorizationSettings") }}</p>
                <div v-if="!updating" @click="addAuthorisationGroup()" class="button-sm text-blue border-blue right">{{ $t("addGroup") }}</div>
            </div>
            <div v-if="!updating">
                <p v-if="allAuthorisationGroups.length > 0" class="text-lg">{{ $t("groups") }}</p>
                <p v-else class="text-lg text-grey">{{ $t("noGroups") }}</p>
                <div class="group wrap vertical-center">
                    <div
                        v-for="(group, groupIndex) in allAuthorisationGroups"
                        :key="'group-' + groupIndex"
                        class="card click"
                        @click="selectedAuthorisationGroup = group"
                        :class="{ 'text-blue border-blue': (group || {})._id == (selectedAuthorisationGroup || {})._id }"
                    >
                        <p>{{ group.name }}</p>
                    </div>
                </div>
                <div v-if="selectedAuthorisationGroup" class="mt">
                    <div class="group vertical-center">
                        <text-field
                            :prop="selectedAuthorisationGroup.name"
                            type="input"
                            inputType="text"
                            :label="$t('name')"
                            @changeValue="newValue => (selectedAuthorisationGroup.name = newValue)"
                        />
                        <div class="group vertical-center right">
                            <div @click="deletedAuthorisationGroups.push(selectedAuthorisationGroup)" class="button-sm text-red border-red">
                                {{ $t("deleteGroup") }}
                            </div>
                            <div @click="addAuthorisation()" class="button-sm text-blue border-blue">{{ $t("addAuthorization") }}</div>
                        </div>
                    </div>
                    <prop-selection :prop="selectedAuthorisationGroup.properties.specials" :values="specialAuthorisations" type="checkbox" cols="3" />
                    <div
                        v-for="(authorisation, authorisationsKey, authorisationsIndex) in selectedAuthorisationGroup.properties.authorisations.properties"
                        :key="'authorisation' + authorisationsKey"
                        class="mv-sm"
                    >
                        <div v-if="editAuthorisation != authorisationsKey" class="badge text-white bg-black">
                            <span>{{ $t("rule") }} {{ authorisationsIndex + 1 }} </span>
                            <span class="text-yellow">{{ authorisation.properties.perm.value }}</span>
                            <i class="fa fa-pen click text-light-blue ml-xxs mr-xs" @click="editAuthorisation = authorisationsKey"></i>
                            <i class="fa fa-times click text-red" @click="removeAuthorisation(authorisationsKey)"></i>
                        </div>
                        <div v-else>
                            <div class="group vertical-center mt">
                                <p>{{ $t("rule") }} {{ authorisationsIndex + 1 }}</p>
                                <text-field
                                    :prop="authorisation.properties.perm.value"
                                    type="input"
                                    inputType="text"
                                    :label="$t('permissions')"
                                    @changeValue="newValue => (authorisation.properties.perm.value = newValue)"
                                />
                            </div>
                            <query-builder
                                :initQuery="authorisation.properties.query.value"
                                @newQuery="
                                    newQuery => {
                                        authorisation.properties.query.value = JSON.stringify(newQuery);
                                    }
                                "
                                :authorisationMode="true"
                            ></query-builder>
                        </div>
                    </div>
                </div>
                <div v-if="this.selectedAuthorisationGroup" class="mt-lg">
                    <p class="text-lg">User</p>
                    <input class="input width-full" type="search" v-model="searchInput" :placeholder="$t('search') + '...'" />
                    <div class="group vertical-center ">
                        <div class="group vertical-center">
                            <div @click="toggleUsersSelection()" class="button-sm" :class="{ 'text-blue border-blue': selectAllUsers }">
                                {{ $t("selectAll") }}
                            </div>
                        </div>
                    </div>
                    <div class="group wrap vertical-center mt-sm">
                        <div
                            v-for="(user, userIndex) in users"
                            :key="'user-' + userIndex"
                            class="card click"
                            @click="toggleUserSelection(user)"
                            :class="{
                                'text-blue border-blue':
                                    (changedUsers[user._id] &&
                                        ((changedUsers[user._id].properties || {}).authorisationGroup || {}).value == (selectedAuthorisationGroup || {})._id) ||
                                    (!changedUsers[user._id] &&
                                        ((user.properties || {}).authorisationGroup || {}).value == (selectedAuthorisationGroup || {})._id)
                            }"
                        >
                            <p
                                class="text-dark-grey text-sm mb-none"
                                :class="{
                                    'text-blue':
                                        (changedUsers[user._id] &&
                                            ((changedUsers[user._id].properties || {}).authorisationGroup || {}).value ==
                                                (selectedAuthorisationGroup || {})._id) ||
                                        (!changedUsers[user._id] &&
                                            ((user.properties || {}).authorisationGroup || {}).value == (selectedAuthorisationGroup || {})._id)
                                }"
                            >
                                {{
                                    (
                                        getAuthorisationGroupById(
                                            (((changedUsers[user._id] || {}).properties || {}).authorisationGroup || {}).value ||
                                                ((user.properties || {}).authorisationGroup || {}).value
                                        ) || {}
                                    ).name || "No Group"
                                }}
                            </p>
                            <p class="text mt-none">{{ user.username }}</p>
                        </div>
                    </div>
                    <div class="group horizontal-center">
                        <div v-if="users.length == usersPage * 20" @click="loadUsers(++usersPage)" class="button-sm text-blue border-blue">
                            {{ $t("loadMore") }}
                        </div>
                    </div>
                </div>
                <div class="group vertical-center mt-sm">
                    <div class="group vertical-center mt-sm right">
                        <div @click="closeModal()" class="button min-width-lg text-red border-red">{{ $t("cancel") }}</div>
                        <div @click="updateAuthorisation()" class="button min-width-lg text-blue border-blue">{{ $t("update") }}</div>
                    </div>
                </div>
            </div>
            <div v-else class="group column horizontal-center mv-xxl">
                <loading-spinner />
                <p class="ml">{{ $t("updating") }} {{ $t("authorizations") }}</p>
            </div>
        </div>
    </div>
</template>

<script>
import TextField from "@/components/TextField";
import QueryBuilder from "@/components/QueryBuilder";
import LoadingSpinner from "@/components/LoadingSpinner";
import PropSelection from "./PropSelection.vue";

/*
    Modal component for managing the authorisations via authorisationsGroups

    Creates/Updats/Deletes authorisationsGroups with authorisations constructed from the QueryBuilder

    user authorisations are updated every time their authorisationsGroups is updated

    Users are link to authorisationsGroups via authorisationGroup property
*/

export default {
    name: "AuthorisationSettings",
    components: {
        "text-field": TextField,
        "query-builder": QueryBuilder,
        "loading-spinner": LoadingSpinner,
        "prop-selection": PropSelection
    },
    data() {
        return {
            specialAuthorisations: [
                { value: "settings", name: "Settings" },
                { value: "authorisationSettings", name: "Authorisation Settings" }
            ],
            preventClose: false,
            users: [],
            initUsers: [],
            usersPage: 1,
            newAuthorisationGroups: [],
            deletedAuthorisationGroups: [],
            authorisationGroups: [],
            initAuthorisationGroups: [],
            authorisationGroupsPage: 1,
            searchInput: "",
            selectedAuthorisationGroup: null,
            changedUsers: {},
            selectAllUsers: false,
            editAuthorisation: null,
            updating: false
        };
    },
    computed: {
        allAuthorisationGroups() {
            let allAuthorisationGroups = [];

            this.newAuthorisationGroups.forEach(group => {
                if (!this.deletedAuthorisationGroups.find(deletedGroup => deletedGroup._id == group._id)) allAuthorisationGroups.push(group);
            });

            this.authorisationGroups.forEach(group => {
                if (!this.deletedAuthorisationGroups.find(deletedGroup => deletedGroup._id == group._id)) allAuthorisationGroups.push(group);
            });

            return allAuthorisationGroups;
        }
    },
    watch: {
        searchInput: {
            handler() {
                this.users = [];

                this.loadUsers();
            }
        }
    },
    methods: {
        closeModal() {
            if (!this.preventClose) {
                this.$store.commit("setBlurPage", false);
                this.$emit("toggled", false);
            } else this.preventClose = false;
        },
        getAuthorisationGroupById(authorisationGroupId) {
            let authorisationGroup = this.allAuthorisationGroups.find(group => group._id == authorisationGroupId);

            return authorisationGroup;
        },
        /** Toggles assignment of user obj in changeUsers */
        toggleUserSelection(user) {
            if (!this.changedUsers[user._id]) this.changedUsers[user._id] = JSON.parse(JSON.stringify(user));

            if (!this.changedUsers[user._id].properties) this.changedUsers[user._id].properties = {};

            if (this.changedUsers[user._id] && (this.changedUsers[user._id].properties.authorisationGroup || {}).value != this.selectedAuthorisationGroup._id) {
                this.changedUsers[user._id].properties.authorisationGroup = {
                    value: this.selectedAuthorisationGroup._id,
                    type: "objectRef"
                };
            } else if (this.changedUsers[user._id]) {
                this.changedUsers[user._id].properties.authorisationGroup.value = null;
            }

            if (
                this.changedUsers[user._id] &&
                (this.changedUsers[user._id].properties.authorisationGroup || {}).value == ((user.properties || {}).authorisationGroup || {}).value
            ) {
                delete this.changedUsers[user._id];
            }
        },
        /** Toggles assignment of all user objs in changeUsers */
        toggleUsersSelection() {
            this.selectAllUsers = !this.selectAllUsers;

            if (this.selectAllUsers) {
                this.users.forEach(user => (this.changedUsers[user._id] = JSON.parse(JSON.stringify(user))));
                Object.keys(this.changedUsers).forEach(userId => {
                    if (!this.changedUsers[userId].properties) this.changedUsers[userId].properties = {};

                    this.changedUsers[userId].properties.authorisationGroup = {
                        value: this.selectedAuthorisationGroup._id,
                        type: "objectRef"
                    };
                });
            } else {
                Object.keys(this.changedUsers).forEach(userId => {
                    this.changedUsers[userId].properties.authorisationGroup = {
                        value: null,
                        type: "objectRef"
                    };
                });
            }

            Object.keys(this.changedUsers).forEach(userId => {
                if (this.changedUsers[userId].properties.authorisationGroup.value == ((this.users[userId].properties || {}).authorisationGroup || {}).value) {
                    delete this.changedUsers[userId];
                }
            });
        },
        generateUID() {
            let firstPart = (Math.random() * 46656) | 0;
            let secondPart = (Math.random() * 46656) | 0;
            firstPart = ("000" + firstPart.toString(36)).slice(-3);
            secondPart = ("000" + secondPart.toString(36)).slice(-3);
            return firstPart + secondPart;
        },
        /** Converts query to authorisation conform query */
        smoothAuthorisations(authorisations) {
            let smoothAuthorisations = authorisations;

            Object.keys(authorisations).forEach(authorisationKey => {
                if (authorisationKey == "$and" || authorisationKey == "$or") {
                    smoothAuthorisations[authorisationKey].forEach((authorisationPart, authorisationPartIndex) => {
                        if (Object.keys(authorisationPart).length == 0) delete smoothAuthorisations[authorisationKey].splice(authorisationPartIndex, 1);
                        else smoothAuthorisations[authorisationKey][authorisationPartIndex] = this.smoothAuthorisations(authorisationPart);
                    });
                } else {
                    if (typeof authorisations[authorisationKey] === "object") {
                        if (authorisations[authorisationKey].hasOwnProperty("$regex")) {
                            if (authorisations[authorisationKey].$regex == "") smoothAuthorisations[authorisationKey].$regex = "(.*?)";
                        }
                    }
                }
            });

            console.log(smoothAuthorisations);

            return smoothAuthorisations;
        },
        addAuthorisationGroup() {
            let uId = this.generateUID();

            this.selectedAuthorisationGroup = {
                _id: uId,
                name: "New Group",
                properties: {
                    authorisations: {
                        properties: {},
                        type: "bag"
                    }
                },
                type: "authorisationGroup"
            };

            this.newAuthorisationGroups.push(this.selectedAuthorisationGroup);
        },
        addAuthorisation() {
            let uId = this.generateUID();

            this.selectedAuthorisationGroup.properties.authorisations.properties[uId] = {
                properties: {
                    query: {
                        value: "",
                        type: "shortText"
                    },
                    perm: {
                        value: "",
                        type: "shortText"
                    },
                    specials: {
                        properties: {},
                        type: "bag"
                    }
                },
                type: "bag"
            };

            this.editAuthorisation = uId;
        },
        removeAuthorisation(authorisationKey) {
            delete this.selectedAuthorisationGroup.properties.authorisations.properties[authorisationKey];
            this.editAuthorisation = Object.keys(this.selectedAuthorisationGroup.properties.authorisations.properties)[0];
        },
        async loadAuthorisationGroups(page = 1) {
            let res = null;
            let query = {
                type: "authorisationGroup"
            };

            try {
                res = await spoo
                    .io()
                    .Objects({ $query: query, $page: page })
                    .get();
            } catch {
                return this.global.errorMessage("Error loading authorisation groups");
            }

            res.forEach(() => {});

            if (res) {
                this.authorisationGroups.push(...res);

                this.initAuthorisationGroups = JSON.parse(JSON.stringify(this.authorisationGroups));

                this.authorisationGroups.forEach(group => {
                    if (!group.properties.specials) {
                        group.properties.specials = {
                            properties: {},
                            type: "bag"
                        };
                    }
                });
            }
        },
        async loadUsers(page = 1) {
            let res = null;
            let query = {
                username: {
                    $regex: this.searchInput,
                    $options: "i"
                }
            };

            try {
                res = await spoo
                    .io()
                    .Users({ $query: query, $page: page })
                    .get();
            } catch {
                return this.global.errorMessage("Error loading users");
            }

            if (res) {
                this.users.push(...res);
                this.initUsers = JSON.parse(JSON.stringify(this.users));
            }
        },
        /** Updates user authorisation & authorisationGroup with an additional standard authorisations */
        async updateUserAuthorisation(initUser, user) {
            let standardAuthorisations = [
                { query: { name: "Pile", type: "pile" }, perm: "r" },
                { query: { name: "Form", type: "card_default_form" }, perm: "r" },
                { query: { name: "Card Default Model", type: "card_default_template" }, perm: "r" },
                { query: { name: "Search Config", type: "search_config" }, perm: "r" },
                { query: { name: "App Config", type: "app_config" }, perm: "r" },
                { query: { role: "eventlog" }, perm: "cr" }
            ];

            if ((((this.selectedAuthorisationGroup.properties.specials || {}).properties || {})["authorisationSettings"] || {}).value == true) {
                standardAuthorisations.push({ query: { type: "authorisationGroup" }, perm: "crud" });
            } else standardAuthorisations.push({ query: { type: "authorisationGroup" }, perm: "r" });

            let updatable = spoo.io().User(user._id);

            let authorisationGroup = this.allAuthorisationGroups.find(group => group._id == user.properties.authorisationGroup.value);
            let newAuthorisation = null;

            if (authorisationGroup) {
                ((user.authorisations || {})[this.$store.state.appId] || []).forEach(authorisation => updatable.removeAuthorisation(authorisation.name));

                standardAuthorisations.forEach(standardAuthorisation => {
                    updatable.setAuthorisation({
                        query: JSON.stringify(standardAuthorisation.query),
                        perm: standardAuthorisation.perm
                    });
                });

                Object.keys(authorisationGroup.properties.authorisations.properties).forEach(authorisationKey => {
                    newAuthorisation = authorisationGroup.properties.authorisations.properties[authorisationKey].properties;

                    newAuthorisation.query.value = JSON.stringify(this.smoothAuthorisations(JSON.parse(newAuthorisation.query.value)));

                    updatable.setAuthorisation({
                        query: JSON.stringify(JSON.parse(newAuthorisation.query.value)) || "{}",
                        perm: newAuthorisation.perm.value.toLowerCase()
                    });
                });
            }

            if (!initUser.properties.authorisationGroup && user.properties.authorisationGroup) {
                updatable.addProperty({ authorisationGroup: user.properties.authorisationGroup });
            } else if (initUser.properties.authorisationGroup.value != user.properties.authorisationGroup.value) {
                updatable.setPropertyValue("authorisationGroup", user.properties.authorisationGroup.value);
            }

            try {
                await updatable.save();
            } catch {
                return this.global.errorMessage("Error updating user authorisations");
            }
        },

        /** Updates authorisationGroups and all users inside these groups */
        async updateAuthorisation() {
            let res = null;
            let tempId = null;
            this.updating = true;

            if (this.deletedAuthorisationGroups) {
                for (let i = 0; i < this.deletedAuthorisationGroups.length; i++) {
                    try {
                        res = await spoo
                            .io()
                            .Object(this.deletedAuthorisationGroups[i]._id)
                            .delete();
                    } catch {
                        return this.global.errorMessage("Error deleting authorisation group");
                    }
                }
            }

            if (this.newAuthorisationGroups) {
                for (let i = 0; i < this.newAuthorisationGroups.length; i++) {
                    tempId = this.newAuthorisationGroups[i]._id;
                    delete this.newAuthorisationGroups[i]._id;

                    try {
                        res = await spoo
                            .io()
                            .Object(this.newAuthorisationGroups[i])
                            .add();
                    } catch {
                        return this.global.errorMessage("Error adding new authorisation group");
                    }

                    this.newAuthorisationGroups[i]._id = res._id;

                    Object.keys(this.changedUsers).forEach(userId => {
                        if ((((this.changedUsers[userId] || {}).properties || {}).authorisationGroup || {}).value == tempId) {
                            this.changedUsers[userId].properties.authorisationGroup.value = res._id;
                        }
                    });

                    for (let j = 0; j < this.users.length; j++) {
                        if (
                            !this.changedUsers[this.users[j]._id] &&
                            (((this.users[j] || {}).properties || {}).authorisationGroup || {}).value == this.newAuthorisationGroups[i]._id
                        ) {
                            await this.updateUserAuthorisation(this.initUsers[j], this.users[j]);
                        }
                    }
                }
            }

            for (let i = 0; i < this.authorisationGroups.length; i++) {
                res = await this.global.updateSpooObj(this.initAuthorisationGroups[i], this.authorisationGroups[i]);

                if (res) {
                    for (let j = 0; j < this.users.length; j++) {
                        if (
                            !this.changedUsers[this.users[j]._id] &&
                            (((this.users[j] || {}).properties || {}).authorisationGroup || {}).value == this.authorisationGroups[i]._id
                        ) {
                            await this.updateUserAuthorisation(this.initUsers[j], this.users[j]);
                        }
                    }
                }
            }

            for (let i = 0; i < this.users.length; i++) {
                if (this.changedUsers[this.users[i]._id]) {
                    await this.updateUserAuthorisation(this.initUsers[i], this.changedUsers[this.users[i]._id]);
                }
            }

            this.updating = false;

            this.closeModal();
        }
    },
    async created() {
        this.loadUsers();
        await this.loadAuthorisationGroups();

        if (this.authorisationGroups) this.selectedAuthorisationGroup = this.authorisationGroups[0];
    }
};
</script>
